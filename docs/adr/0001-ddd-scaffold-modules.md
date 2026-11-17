# 后端使用 7 模块 DDD 脚手架

领域只有出版物、借阅、读者等少数概念，单模块按功能分包完全够用；但后端仍用 `create-ddd-project` 脚手架（ddd-scaffold-std-jdk17 1.8）生成 api、app、case、domain、trigger、infrastructure、types 七个模块，以沿用团队熟悉的分层和 DDD 编码规范，让借阅规则与持久化、接口层清晰隔离。脚手架生成后立即升级到 Java 21 和 Spring Boot 4.1.x，并移除 fastjson、dom4j、xstream、jjwt 等用不到或有已知漏洞的依赖。
