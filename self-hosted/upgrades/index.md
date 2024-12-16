---
标题: 升级 TimescaleDB
摘要: 对已安装的自托管 TimescaleDB 进行原地升级。
产品: [自托管]
关键词: [升级]
---

import ConsiderCloud from "versionContent/_partials/_consider-cloud.mdx";

# 升级 TimescaleDB

**主要版本升级**是指您从TimescaleDB `X.<次要版本>`更新到`Y.<次要版本>`。
**次要版本升级**是指您从TimescaleDB `<主版本>.x`更新到TimescaleDB `<主版本>.y`。
您在原地升级自行托管的TimescaleDB安装。

<ConsiderCloud />

本节向您展示如何：

* 将自行托管的TimescaleDB升级到新的[次要版本][upgrade-minor]。
* 将自行托管的TimescaleDB升级到新的[主要版本][upgrade-major]。
* 将运行在[Docker容器][upgrade-docker]中的自行托管TimescaleDB升级到新的次要版本。
* 将[PostgreSQL][upgrade-pg]升级到新的版本。
* 将自行托管的TimescaleDB降级到[之前的次要版本][downgrade]。

[downgrade]: /self-hosted/:currentVersion:/upgrades/downgrade/
[upgrade-docker]: /self-hosted/:currentVersion:/upgrades/upgrade-docker/
[upgrade-major]: /self-hosted/:currentVersion:/upgrades/major-upgrade/
[upgrade-minor]: /self-hosted/:currentVersion:/upgrades/minor-upgrade/
[upgrade-pg]: /self-hosted/:currentVersion:/upgrades/upgrade-pg/
[upgrade-tshoot]: /self-hosted/:currentVersion:/troubleshooting/

