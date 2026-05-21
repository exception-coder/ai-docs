# Python AI 服务架构（编码摘要）

> 配套：[Python AI 服务架构-current.md](Python AI 服务架构-current.md)
> 定位：项目级基础设施落地清单

## 0. 一句话设计结论

新建 `python-services/ai-vision/` 统一 FastAPI 服务（端口 9501），架构 = `server.py + registry.py + models/{name}.py` 三件套；模型按需懒加载；Java 侧 `AiVisionClient` 走 HTTP/multipart 调用。本期挂一个模型 `face_age`（MiVOLO），未来按"加一个 `models/x.py` + `server.py` import"的标准流程扩。

## 1. 核心业务规则

- 服务端口 **9501**（faster-whisper 是 9500）
- 框架：FastAPI + uvicorn
- 模型生命周期：config.yaml 决定启动时加载哪些；空闲 30 分钟自动卸载（idle_unload）
- 单进程串行推理，不做模型间并发
- 临时文件 finally 删；启动时清理 tmpdir
- start.bat **注释强制 ASCII**（GBK 编码陷阱），中文说明放 README.md
- 模型从 HuggingFace 自动下载，提示用户走 HF_ENDPOINT mirror + HTTPS_PROXY
- Java 侧 isHealthy() 缓存 60s；调用前 lazy check
- 服务未启动时 Java 端清晰报错"请双击 start.bat"，不做自动拉起

## 2. 接口入口

### Python 服务端

| Method | Path | 说明 |
|--------|------|------|
| GET | `/health` | 健康检查 + 模型加载状态 |
| GET | `/models` | 全部已注册模型 + 当前状态 |
| POST | `/{model}/load` | 手动加载模型 |
| POST | `/{model}/unload` | 手动卸载模型 |
| POST | `/face-age/analyze` | MiVOLO 人物年龄识别（业务端点示例） |

### Java 端入口

`com.exceptioncoder.toolbox.treesize.client.AiVisionClient` 暴露同等粒度的方法。

## 3. 涉及文件清单

### 3.1 Python 服务新建文件

#### `python-services/ai-vision/server.py`

```python
"""kai-toolbox ai-vision: 统一视觉 AI 模型服务。

各模型挂载到 /{model}/{action} 路由（详见 models/*.py）。
模型生命周期由 registry 管理：lazy load + idle unload，避免显存占用过多。

启动：双击 start.bat（Windows）或 ./start.sh（Linux/macOS）
中文说明详见 README.md（start.bat 的注释只能 ASCII，避免 GBK 解析陷阱）。
"""
from __future__ import annotations

import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI, HTTPException

from registry import ModelRegistry
from models import face_age   # 未来扩模型时在此 import

logging.basicConfig(level=logging.INFO,
                    format="%(asctime)s %(levelname)-5s [%(name)s] %(message)s")
log = logging.getLogger("ai-vision")

registry = ModelRegistry(config_path="config.yaml")


@asynccontextmanager
async def lifespan(_app: FastAPI):
    # 启动：自动加载 config.yaml 里 enabled=true 的模型
    registry.startup()
    log.info("ai-vision ready: models=%s", registry.health_view())
    yield
    # 关停：释放所有模型显存
    registry.shutdown()


app = FastAPI(title="kai-toolbox ai-vision", lifespan=lifespan)

# 业务端点注册（每个模型一个 register 函数）
face_age.register(app, registry)


@app.get("/health")
def health(): return {"status": "ok", "models": registry.health_view()}

@app.get("/models")
def list_models(): return registry.list()

@app.post("/{name}/load")
def load_model(name: str):
    try: registry.load(name)
    except KeyError: raise HTTPException(404, f"unknown model: {name}")
    return {"loaded": name}

@app.post("/{name}/unload")
def unload_model(name: str):
    try: registry.unload(name)
    except KeyError: raise HTTPException(404, f"unknown model: {name}")
    return {"unloaded": name}
```

#### `python-services/ai-vision/registry.py`

```python
"""模型生命周期管理：懒加载 / 卸载 / 空闲 GC / 健康视图。

不依赖具体模型实现，所有模型通过 register_loader 注入 _load() 闭包。
"""
import logging
import threading
import time
from typing import Callable

import yaml

log = logging.getLogger("ai-vision.registry")


class ModelRegistry:
    def __init__(self, config_path: str):
        with open(config_path, encoding="utf-8") as f:
            self.config = yaml.safe_load(f)
        self.idle_unload_minutes = self.config["service"].get("idle_unload_minutes", 30)
        # name -> { "loader": callable, "instance": obj_or_none, "last_used": ts }
        self._slots: dict[str, dict] = {}
        self._lock = threading.RLock()
        self._gc_thread: threading.Thread | None = None

    def register_loader(self, name: str, loader: Callable[[], object]):
        """模型在 server.py 启动时调用，注册自己的加载函数。"""
        cfg = self.config["models"].get(name, {})
        with self._lock:
            self._slots[name] = {"loader": loader, "instance": None,
                                  "config": cfg, "last_used": 0.0}

    def startup(self):
        """按 config.enabled=true 加载初始模型；起 GC 线程。"""
        for name, cfg in self.config["models"].items():
            if cfg.get("enabled") and name in self._slots:
                self.load(name)
        if self.idle_unload_minutes:
            self._gc_thread = threading.Thread(target=self._gc_loop, daemon=True,
                                                name="ai-vision-gc")
            self._gc_thread.start()

    def shutdown(self):
        with self._lock:
            for name in list(self._slots.keys()):
                if self._slots[name]["instance"] is not None:
                    self.unload(name)

    def load(self, name: str):
        with self._lock:
            slot = self._slots.get(name)
            if slot is None: raise KeyError(name)
            if slot["instance"] is not None:
                slot["last_used"] = time.time()
                return  # 幂等
            log.info("loading model: %s", name)
            t0 = time.time()
            slot["instance"] = slot["loader"]()
            slot["last_used"] = time.time()
            log.info("loaded model: %s in %.1fs", name, time.time() - t0)

    def unload(self, name: str):
        with self._lock:
            slot = self._slots.get(name)
            if slot is None: raise KeyError(name)
            if slot["instance"] is None: return  # 幂等
            log.info("unloading model: %s", name)
            slot["instance"] = None
            # 强制 GC + cuda empty_cache
            import gc; gc.collect()
            try:
                import torch
                if torch.cuda.is_available(): torch.cuda.empty_cache()
            except ImportError: pass

    def get(self, name: str):
        """业务端点调：未加载抛 HTTPException(503) 提示用户先 /load。"""
        from fastapi import HTTPException
        with self._lock:
            slot = self._slots.get(name)
            if slot is None: raise HTTPException(404, f"unknown model: {name}")
            if slot["instance"] is None:
                # 默认行为：自动按需加载
                self.load(name)
                slot = self._slots[name]
            slot["last_used"] = time.time()
            return slot["instance"]

    def health_view(self):
        with self._lock:
            return {n: ("loaded" if s["instance"] else "unloaded")
                    for n, s in self._slots.items()}

    def list(self):
        with self._lock:
            return [{"name": n,
                     "status": "loaded" if s["instance"] else "unloaded",
                     "config": s["config"],
                     "last_used": s["last_used"]}
                    for n, s in self._slots.items()]

    def _gc_loop(self):
        threshold_s = self.idle_unload_minutes * 60
        while True:
            time.sleep(60)
            now = time.time()
            for name in list(self._slots.keys()):
                with self._lock:
                    slot = self._slots[name]
                    if slot["instance"] is None: continue
                    if now - slot["last_used"] > threshold_s:
                        log.info("idle %.0fs > %ds, unloading: %s",
                                 now - slot["last_used"], threshold_s, name)
                        self.unload(name)
```

#### `python-services/ai-vision/models/__init__.py`

空文件即可（标识 package）。

#### `python-services/ai-vision/models/face_age.py`

```python
"""MiVOLO 人物年龄 + 性别识别。

输入：单张图片（jpg / png / 任意 PIL 可读格式）
输出：检测到的人物列表 + 主体（最大 bbox）
"""
import io
import logging
from typing import Any

import numpy as np
from fastapi import FastAPI, UploadFile, File, HTTPException
from PIL import Image

log = logging.getLogger("ai-vision.face_age")
MODEL_NAME = "face_age"

# 年龄段切分（与 Java 端 PersonAgeGroup 枚举保持一致）
def _age_group(age: float) -> str:
    if age < 3:   return "infant"
    if age < 13:  return "child"
    if age < 20:  return "teen"
    if age < 36:  return "young_adult"
    if age < 56:  return "middle_age"
    return "senior"


def _load():
    """加载 MiVOLO 模型 + 内嵌的 YOLOv8 检测器。"""
    # 关键导入延迟到加载时，避免 server.py import 时全部加载
    from mivolo.predictor import Predictor
    # config 通过 registry 注入到这里：用 closure 不太优雅，从环境变量读更稳
    import os
    checkpoint = os.getenv("MIVOLO_CHECKPOINT", "./models/checkpoints/mivolo_d1.pth.tar")
    detector = os.getenv("MIVOLO_DETECTOR", "./models/checkpoints/yolov8x_person_face.pt")
    log.info("loading MiVOLO checkpoint=%s detector=%s", checkpoint, detector)
    return Predictor(detector_weights=detector,
                     checkpoint=checkpoint,
                     device="cuda",
                     half=True,
                     disable_faces=False,
                     with_persons=True)


def register(app: FastAPI, registry):
    registry.register_loader(MODEL_NAME, _load)

    @app.post("/face-age/analyze")
    async def analyze(file: UploadFile = File(...)):
        try:
            img_bytes = await file.read()
            img = Image.open(io.BytesIO(img_bytes)).convert("RGB")
            arr = np.array(img)
        except Exception as e:
            raise HTTPException(400, f"invalid image: {e}")

        predictor = registry.get(MODEL_NAME)
        try:
            # MiVOLO Predictor 的实际接口需对照 README；这里给出推断后处理骨架
            detected_objects, _annotated_img = predictor.recognize(arr)
        except Exception as e:
            log.exception("MiVOLO inference failed")
            raise HTTPException(500, f"inference failed: {e}")

        people = []
        for obj in detected_objects:
            # 假设 obj 提供 bbox / gender / age / confidence；以 MiVOLO 实际 API 为准
            people.append({
                "bbox": list(obj.bbox),
                "gender": obj.gender,                  # 'M' / 'F'
                "gender_confidence": float(obj.gender_score),
                "age": float(obj.age),
                "age_confidence": float(obj.age_score),
            })

        # 主体：选 bbox 面积最大的一个；若无人物 main_person=None
        main = None
        if people:
            largest = max(people, key=lambda p: p["bbox"][2] * p["bbox"][3])
            main = {
                "gender": largest["gender"],
                "age": int(round(largest["age"])),
                "age_group": _age_group(largest["age"]),
                "confidence": min(largest["age_confidence"], largest["gender_confidence"]),
            }

        return {
            "people": people,
            "main_person": main,
            "model_version": "mivolo-v2",
        }
```

> **注意**：上面的 `predictor.recognize(arr)` 是按 MiVOLO 仓库示例草拟的；实际 API 以 `pip install` 后的 `mivolo.predictor` 模块为准。本文档的目标是给出**集成骨架**，预计 1-2 处方法名要根据实际版本微调。

#### `python-services/ai-vision/config.yaml`

```yaml
# kai-toolbox ai-vision 配置
service:
  host: "127.0.0.1"
  port: 9501
  idle_unload_minutes: 30        # 空闲 30 分钟自动卸载；null = 禁用 GC

models:
  face_age:
    enabled: true                # 启动时自动加载
```

#### `python-services/ai-vision/requirements.txt`

```text
# Core HTTP
fastapi>=0.110.0
uvicorn[standard]>=0.27.0
python-multipart>=0.0.9
pyyaml>=6.0

# Imaging
Pillow>=10.0.0
numpy>=1.26.0
opencv-python>=4.9.0

# PyTorch (CUDA 12.x 默认；CPU only 用户改用 --index-url cpu)
torch>=2.2.0
torchvision>=0.17.0
ultralytics>=8.2.0     # MiVOLO 依赖 YOLOv8

# MiVOLO 直装（git 仓库）
mivolo @ git+https://github.com/WildChlamydia/MiVOLO.git
```

> 部分用户的 pip 可能无法直接从 GitHub 装包，备选是 git clone 后 `pip install -e ./MiVOLO`。

#### `python-services/ai-vision/start.bat`

仿照 faster-whisper 的格式，**注释强制 ASCII**：

```batch
@echo off
chcp 65001 > nul
REM ---------------------------------------------------------------------------
REM IMPORTANT: Keep all REM comments in this file ASCII-only.
REM Same GBK-encoding caveat as faster-whisper/start.bat. See README.md for
REM the Chinese explanation.
REM ---------------------------------------------------------------------------
REM Windows launcher for the ai-vision service.
REM First run: creates .venv + pip install (PyTorch+CUDA ~ 5-10 min download).
REM Subsequent runs: < 20 sec (MiVOLO load dominated by disk).
REM
REM Override port / checkpoints via env vars BEFORE running:
REM   set AI_VISION_PORT=9501
REM   set MIVOLO_CHECKPOINT=D:\models\mivolo_d1.pth.tar
REM   set MIVOLO_DETECTOR=D:\models\yolov8x_person_face.pt
REM   set HTTPS_PROXY=http://127.0.0.1:7897
REM   set HF_ENDPOINT=https://hf-mirror.com

cd /d %~dp0

if not exist .venv (
    echo [setup] creating venv...
    python -m venv .venv
    if errorlevel 1 (
        echo [setup] failed to create venv. Need Python 3.10+ in PATH.
        exit /b 1
    )
)

call .venv\Scripts\activate.bat

echo [setup] installing/upgrading dependencies (this may take a while on first run)...
pip install -q --upgrade pip
pip install -q -r requirements.txt
if errorlevel 1 (
    echo [setup] pip install failed.
    exit /b 1
)

if not defined AI_VISION_PORT set AI_VISION_PORT=9501

echo [start] port=%AI_VISION_PORT%
if defined HTTPS_PROXY echo [start] HTTPS_PROXY=%HTTPS_PROXY%
if defined HF_ENDPOINT echo [start] HF_ENDPOINT=%HF_ENDPOINT%
echo [start] uvicorn at http://127.0.0.1:%AI_VISION_PORT%
python -m uvicorn server:app --host 127.0.0.1 --port %AI_VISION_PORT% --log-level info
```

#### `python-services/ai-vision/README.md`

中文说明文档，包含：
- 服务作用 + 默认端口 9501
- 首次部署：双击 start.bat
- 模型 checkpoint 下载提示（HuggingFace + mirror）
- 常见 troubleshooting：CUDA 版本不匹配 / proxy / 端口占用
- 现有端点清单
- 如何扩展新模型（指向本设计文档）

#### `python-services/ai-vision/.gitignore`

```
.venv/
__pycache__/
*.pyc
checkpoints/
*.pth
*.pth.tar
*.pt
```

### 3.2 Java 端新建文件

#### `com.exceptioncoder.toolbox.treesize.client.AiVisionProperties`

```java
@ConfigurationProperties(prefix = "toolbox.ai-vision")
@Component
public class AiVisionProperties {
    private String url = "http://127.0.0.1:9501";
    private int timeoutSeconds = 60;
    private boolean healthCheckOnStartup = false;
    // getter / setter ...
}
```

#### `com.exceptioncoder.toolbox.treesize.client.dto.HealthView`

```java
public record HealthView(String status, Map<String, String> models) {}
```

#### `com.exceptioncoder.toolbox.treesize.client.dto.FaceAgeResult`

```java
public record FaceAgeResult(
    List<Person> people,
    MainPerson mainPerson,
    String modelVersion
) {
    public record Person(double[] bbox, String gender, double genderConfidence,
                          double age, double ageConfidence) {}
    public record MainPerson(String gender, int age, String ageGroup, double confidence) {}
}
```

#### `com.exceptioncoder.toolbox.treesize.client.AiVisionClient`

```java
@Component
public class AiVisionClient {

    private static final Logger log = LoggerFactory.getLogger(AiVisionClient.class);

    private final RestClient http;
    private final AiVisionProperties props;
    private final AtomicReference<HealthCheckCache> healthCache =
        new AtomicReference<>(new HealthCheckCache(0L, false));

    public AiVisionClient(AiVisionProperties props) {
        this.props = props;
        this.http = RestClient.builder()
            .baseUrl(props.getUrl())
            .requestFactory(timeoutFactory(props.getTimeoutSeconds()))
            .build();
    }

    /** Java 业务调用前先打这个。缓存 60s 避免每次推断前都打 /health。 */
    public boolean isHealthy() {
        HealthCheckCache c = healthCache.get();
        if (System.currentTimeMillis() - c.checkedAt() < 60_000) return c.healthy();
        boolean ok;
        try {
            HealthView v = http.get().uri("/health").retrieve().body(HealthView.class);
            ok = v != null && "ok".equals(v.status());
        } catch (Exception e) {
            ok = false;
        }
        healthCache.set(new HealthCheckCache(System.currentTimeMillis(), ok));
        return ok;
    }

    public FaceAgeResult analyzeFaceAge(byte[] imageBytes, String filename) {
        if (!isHealthy()) {
            throw new IllegalStateException(
                "ai-vision 服务未启动。请双击 python-services/ai-vision/start.bat。");
        }
        MultiValueMap<String, HttpEntity<?>> parts = new MultipartBodyBuilder()
            .part("file", new ByteArrayResource(imageBytes) {
                @Override public String getFilename() { return filename; }
            })
            .build();
        return http.post()
            .uri("/face-age/analyze")
            .contentType(MediaType.MULTIPART_FORM_DATA)
            .body(parts)
            .retrieve()
            .body(FaceAgeResult.class);
    }

    private record HealthCheckCache(long checkedAt, boolean healthy) {}
}
```

### 3.3 application.yml 新增

```yaml
toolbox:
  # ... 已有配置 ...
  ai-vision:
    url: ${AI_VISION_URL:http://127.0.0.1:9501}
    timeout-seconds: 60
    health-check-on-startup: false
```

## 4. 数据结构

无新数据库表。视频表的人物年龄字段定义在「视频人物年龄识别」子模块文档（下期），不在此文档落地。

## 5. 重要约束与边界

| 约束 | 说明 |
|------|------|
| **服务独立运行** | 用户手动启 start.bat；Java 不负责拉起 |
| **端口隔离** | 9501（ai-vision） vs 9500（faster-whisper） |
| **start.bat ASCII** | 注释绝不能写中文，GBK 解析陷阱 |
| **依赖隔离** | 独立 .venv，不与 faster-whisper 共用 |
| **模型 checkpoint** | 用环境变量覆盖默认路径；新机部署默认从 HuggingFace 下载 |
| **health 缓存 60s** | 减少 /health 往返；过期重新打 |
| **新模型扩展四步** | (1)写 models/x.py (2)server.py import (3)config.yaml 加项 (4)重启 |
| **业务循环不进 Python** | Java 端 ProcessingJobService 跑循环；Python 服务只处理单图 |
| **超时** | Java 端 60s 单请求；MiVOLO 单图 < 2s，余量充足 |

## 6. 测试要点

- **手动启动验证**：双击 start.bat，console 看到 "[start] uvicorn at..."，浏览器 `/docs` 应能打开 Swagger UI
- **/health**：curl 应返 `{"status":"ok","models":{"face_age":"loaded"}}`
- **/face-age/analyze**：curl -F file=@test.jpg → 返回 JSON
- **未启动场景**：停服务后 Java 端调 analyzeFaceAge → 抛 IllegalStateException 含明确指引
- **空闲卸载**：日志看 30 分钟后自动 unload，再调一次 → 自动 reload
- **大图**：MB 级 jpg 上传，60s 内返回
- **错误图**：上传 txt 文件 → 400 invalid image
- **未注册模型**：调 `/foo/load` → 404 unknown model

## 7. 部署 troubleshooting（写入 README.md）

| 症状 | 排查 |
|------|------|
| `[setup] failed to create venv` | 装 Python 3.10+，加入 PATH |
| `torch.cuda.is_available() == False` | 装 CUDA 12.x + 对应 NVIDIA 驱动；CPU 用户改 cpu-only torch（性能很差） |
| HuggingFace 下载超时 | `set HF_ENDPOINT=https://hf-mirror.com` 或 `set HTTPS_PROXY=http://127.0.0.1:7897` |
| 端口 9501 被占 | `set AI_VISION_PORT=9502` 后再启 |
| MiVOLO checkpoint 路径错 | 设置 `MIVOLO_CHECKPOINT` / `MIVOLO_DETECTOR` 环境变量 |
| Java 端调用失败 | 先 `curl http://127.0.0.1:9501/health` 验证服务存活 |

## 8. 后续模型扩展示例（未来工作占位，不在本期）

```python
# python-services/ai-vision/models/scene_class.py
MODEL_NAME = "scene_class"

def _load():
    from transformers import pipeline
    return pipeline("image-classification", model="microsoft/resnet-50", device=0)

def register(app, registry):
    registry.register_loader(MODEL_NAME, _load)

    @app.post("/scene/classify")
    async def classify(file: UploadFile = File(...)):
        ...
```

在 `server.py` 加 `from models import scene_class; scene_class.register(app, registry)`；`config.yaml` 加 `scene_class: { enabled: false }`（按需启）；Java 加 `AiVisionClient.classifyScene(...)`。
