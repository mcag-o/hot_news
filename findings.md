# 调研发现

## 项目概况
- 待确认：从文件结构看，这是一个 Python 项目，包含 API、爬虫站点适配器、分析模块、Telegram 机器人与配置文件。
- 入口候选文件包括 `run.py`、`app/main.py`、`tg_bot.py`。

## 初步关注点
- `requirements.txt` 决定依赖是否完整。
- `config/config.yaml` 可能决定服务是否可启动。
- `app/core/config.py`、`app/core/db.py`、`app/db/mysql.py`、`app/db/redis.py` 可揭示外部服务要求。
- `test/` 下测试可用于快速判断“能否跑起来”。

## 已确认事实
- 项目主形态是一个 FastAPI 服务，入口为 `run.py` 加载配置后启动 `app.main:app`。
- `app/main.py` 在应用生命周期启动时会立即初始化 MySQL 与 Redis，并启动后台爬虫线程；因此默认不是“纯静态 API”，而是强依赖外部服务的后端进程。
- 项目还包含独立的 Telegram 机器人脚本 `tg_bot.py`，但该脚本依赖线上接口 `https://orz.ai/api/v1/dailynews/`，并不直接调用本地应用。
- 默认配置文件 `config/config.yaml` 写死了本地 MySQL (`news_crawler`) 与 Redis (`localhost:6379`) 连接参数，因此在未准备数据库和缓存时，应用启动大概率失败。
- README 展示了线上地址与公开 API，但这些说明不能直接证明当前仓库在本地开箱即用。

## 可用性风险线索
- `app/core/db.py` 使用原生 `pymysql` 并直接查询 `news` 表，说明数据库不但要可连接，还要预先存在正确表结构。
- `app/db/mysql.py` 同时从 `.models` 导入 `News`，随后又在本文件重新定义同名类，存在设计混乱迹象。
- `requirements.txt` 包含 `selenium` 与 `webdriver-manager`，说明部分抓取能力可能依赖浏览器环境。
- `app/services/__init__.py` 在模块导入时就会创建并启动 APScheduler，意味着仅仅导入服务层也会触发调度器初始化。
- `app/services/crawler.py` 的异常处理中引用了未在异常路径稳定定义的 `date_str`，超时或任务级异常分支存在潜在二次报错风险。
- `app/core/cache.py` 的 `init_cache()` 连接 Redis 失败时只记录日志不抛出异常，因此服务可能在“缓存不可用”的退化状态下继续运行，但接口读取缓存时会持续失败。
- `test/crawler_test.py` 中导入路径写成 `app.service.sites`，与实际目录 `app/services/sites` 不一致，表明至少有一份测试是失效的。
- README 中机器人启动命令写的是 `python3 news_tg_bot.py`，而仓库实际文件名是 `tg_bot.py`，文档与代码不一致。
- README 中列出的部分平台代码与源码实现命名不一致，例如文档出现 `shaoshupai` / `36kr` / `52pojie` / `v2ex`，源码与目录实际更接近 `sspai` / `tskr` / `ftpojie` / `vtex`，存在接口文档漂移。

## 验证结果
- 系统 Python 环境下执行测试失败，因为未安装 `pytest`，说明仓库未提供完整开发/测试环境自举能力。
- 创建虚拟环境并按 `requirements.txt` 安装依赖后，`app/core/config.py` 可成功加载默认配置，证明配置文件格式本身有效。
- 同一虚拟环境中导入 `run.py` / `app.main` 失败，根因是 `fastapi 0.125.0` 与 `pydantic 1.10.26` 组合不兼容：FastAPI 运行时需要 `pydantic.TypeAdapter`（Pydantic v2 特性）。
- 运行 `test/hackernews_test.py` 时又出现 `SQLAlchemy 2.0.23` 在 Python 3.14 下的断言错误，显示当前环境与依赖版本至少部分不兼容。
- `python -m py_compile` 可通过，说明仓库代码大体无明显语法错误，但运行时依赖和环境适配未完成。

## 初步结论
- 该仓库不像是“假的项目”；它具备真实项目常见结构：可配置 API 服务、站点抓取器、缓存/数据库接入、分析模块、通知与机器人集成。
- 但“当前仓库在本地真实可用”这一点无法成立；在默认环境下，它尚不能稳定完成启动与验证。
- 更准确的判断是：这是一个真实业务方向的半成品/历史项目仓库，具有真实代码与线上痕迹，但当前可维护性、环境锁定与本地复现能力不足。

## 过程异常
- planning-with-files 技能建议的会话恢复脚本路径不存在，已改为手动建立计划文件。