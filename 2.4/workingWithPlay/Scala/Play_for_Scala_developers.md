#Play Scala开发

Play应用程序开发的 Scala API 都放在 `play.api` 包中。

> API中直接放在`play`包下面的(如 `play.mvc`)是留给Java开发者的。作为Scala开发者, 只需要看 `play.api.mvc`。

##主要概念
1. HTTP编程
    1. [Actions, Controllers 和 Results](Main_concepts/01_HTTP_programming/01_Actions_Controllers_and_Results.md)
    2. [HTTP 路由](Main_concepts/01_HTTP_programming/02_HTTP_Routing.md)
    3. [操纵results](Main_concepts/01_HTTP_programming/03_Manipulating_results.md)
    4. [Session 和 Flash scopes](Main_concepts/01_HTTP_programming/04_Session_and_Flash_scopes.md)
    5. [请求体解析器](Main_concepts/01_HTTP_programming/05_Body_parsers.md)
    6. [Actions 构成](Main_concepts/01_HTTP_programming/06_Actions_composition.md)
    7. [内容协商](Main_concepts/01_HTTP_programming/07_Content_negotiation.md)
    8. [处理错误](Main_concepts/01_HTTP_programming/08_Handling_errors.md)
2. 异步HTTP编程
    1. [异步results](Main_concepts/02_Asynchronous_HTTP_programming/01_Asynchronous_results.md)
    2. [Streaming HTTP 响应](Main_concepts/02_Asynchronous_HTTP_programming/02_Streaming_HTTP_responses.md)
    3. [Comet sockets](Main_concepts/02_Asynchronous_HTTP_programming/03_Comet_sockets.md)
    4. [WebSockets](Main_concepts/02_Asynchronous_HTTP_programming/04_WebSockets.md)
3. 模板引擎
    1. [Scala 模板语法](Main_concepts/03_The_template_engine/01_Scala_templates_syntax.md)
    2. [常见使用案例](Main_concepts/03_The_template_engine/02_Common_use_cases.md)
    3. [自定义格式](Main_concepts/03_The_template_engine/03_Custom_format.md)
4. 表单提交和验证
    1. [处理表单提交](Main_concepts/04_Form_submission_and_validation/01_Handling_form_submission.md)
    2. [防范 CSRF](Main_concepts/04_Form_submission_and_validation/02_Protecting_against_CSRF.md)
    3. [自定义验证器](Main_concepts/04_Form_submission_and_validation/03_Custom_Validations.md)
    4. [自定义表单域构造器](Main_concepts/04_Form_submission_and_validation/04_Custom_Field_Constructors.md)
5. 处理Json
    1. [JSON 基础](Main_concepts/05_Working_with_Json/01_JSON_basics.md)
    2. [JSON 与 HTTP](Main_concepts/05_Working_with_Json/02_JSON_with_HTTP.md)
    3. [JSON Reads/Writes/Format Combinators](Main_concepts/05_Working_with_Json/03_JSON_Reads_Writes_Format_Combinators.md)
    4. [JSON Transformers](Main_concepts/05_Working_with_Json/04_JSON_Transformers.md)
    5. [JSON Macro Inception](Main_concepts/05_Working_with_Json/05_JSON_Macro_Inception.md)
6. [处理XML](Main_concepts/06_Working_with_XML.md)
7. [处理文件上传](Main_concepts/07_Handling_file_upload.md)
8. 访问SQL数据库
    1. [配置和使用JDBC](Main_concepts/08_Accessing_an_SQL_database/01_Configuring_and_using_JDBC.md)
    2. 使用Slick访问你的数据库
        1. [使用Play Slick](Main_concepts/08_Accessing_an_SQL_database/02_01_Using_Play_Slick.md)
        2. [Play Slick迁移指南](Main_concepts/08_Accessing_an_SQL_database/02_02_Play_Slick_migration_guide.md)
        3. [Play Slick高级应用](Main_concepts/08_Accessing_an_SQL_database/02_03_Play_Slick_advanced_topics.md)
        4. [Play Slick问题集](Main_concepts/08_Accessing_an_SQL_database/02_04_Play_Slick_FAQ.md)
    3. [使用Anorm访问你的数据库](Main_concepts/08_Accessing_an_SQL_database/03_Using_Anorm_to_access_your_database.md)
    4. [与其它数据库访问库集成](Main_concepts/08_Accessing_an_SQL_database/04_Integrating_with_other_database_access_libraries.md)
9. [使用缓存](Main_concepts/09_Using_the_Cache.md)
10. 调用WebServices
    1. [Play WS API](Main_concepts/10_Calling_WebServices/01_The_Play_WS_API.md)
    2. [连接到OpenID服务](Main_concepts/10_Calling_WebServices/02_Connecting_to_OpenID_services.md)
    3. [通过OAuth进行访问资源保护](Main_concepts/10_Calling_WebServices/03_Accessing_resources_protected_by_OAuth.md)
11. [集成Akka](Main_concepts/11_Integrating_with_Akka.md)
12. [国际化](Main_concepts/12_Internationalization.md)
13. 测试应用程序
    1. [测试你的应用程序](Main_concepts/13_Testing_your_application/01_Testing_your_Application.md)
    2. [用ScalaTest测试](Main_concepts/13_Testing_your_application/02_Testing_with_ScalaTest.md)
    3. [用ScalaTest编写功能测试](Main_concepts/13_Testing_your_application/03_Writing_functional_tests_with_ScalaTest.md)
    4. [用specs2测试](Main_concepts/13_Testing_your_application/04_Testing_with_specs2.md)
    5. [用specs2编写功能测试](Main_concepts/13_Testing_your_application/05_Writing_functional_tests_with_specs2.md)
    6. [用Guice测试](Main_concepts/13_Testing_your_application/06_Testing_with_Guice.md)
    7. [用数据库测试](Main_concepts/13_Testing_your_application/07_Testing_with_databases.md)
    8. [测试web服务端](Main_concepts/13_Testing_your_application/08_Testing_web_service_clients.md)
14. [日志](Main_concepts/14_Logging.md)

##高级应用
1. [响应式处理数据流](Advanced_topics/01_Handling_data_streams_reactively/Handling_data_streams_reactively.md)
    1. [Iteratees](Advanced_topics/01_Handling_data_streams_reactively/01_Iteratees.md)
    2. [Enumerators](Advanced_topics/01_Handling_data_streams_reactively/02_Enumerators.md)
    3. [Enumeratees](Advanced_topics/01_Handling_data_streams_reactively/03_Enumeratees.md)
2. HTTP 架构
    1. [HTTP API](Advanced_topics/02_HTTP_architecture/01_HTTP_API.md)
    2. [HTTP 过滤器](Advanced_topics/02_HTTP_architecture/02_HTTP_filters.md)
    3. [HTTP 请求处理程序](Advanced_topics/02_HTTP_architecture/03_HTTP_request_handlers.md)
3. 依赖注入
    1. [运行时依赖注入](Advanced_topics/03_Dependency_injection/01_Runtime_dependency_injection.md)
    2. [编译时依赖项注入](Advanced_topics/03_Dependency_injection/02_Compile_time_dependency_injection.md)
4. 高级路由
    1. [字符串 Interpolating 路由 DSL](Advanced_topics/04_Advanced_routing/01_String_Interpolating_Routing_DSL.md)
    2. [Javascript 路由](Advanced_topics/04_Advanced_routing/02_Javascript_routing.md)
5. [扩展Play](Advanced_topics/05_Extending_Play.md)
6. [嵌入Play](Advanced_topics/06_Embedding_Play.md)