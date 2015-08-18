#ʹ��Play����̨

##��������̨
Play����̨��һ������sbt�Ŀ�������̨����������PlayӦ�ó���������������ڽ��й���

Ҫ����Play����̨, �л�����Ŀ���ڵ�Ŀ¼, Ȼ������Activator:

```shell
$ cd my-first-app
$ activator
```

![](console.png)

##��ð���
ʹ��`help` ������Ի���йؿ�������Ļ�����������Ҳ�����ں������һ���ض����������ƣ��Ի������������ذ�����Ϣ��

```shell
[my-first-app] $ help run
```

##�ڿ���ģʽ�����з���
Ҫ����ǰӦ�ó��������ڿ���ģʽ��, ʹ��`run`����:

```shell
[my-first-app] $ run
```

![](consoleRun.png)

�����ģʽ��, �������������Զ�ˢ�¹���, ��������ÿһ������Play�����������Ŀ�����±���Դ���롣������Ҫ��Ӧ�ó���Ҳ���Զ�������

����б�����������������������ֱ�ӿ����������Ľ��:

![](errorPage.png)

Ҫֹͣ������, ���� Crtl+D ��, Ȼ��ͻ᷵��Play����̨��

##����
��Play����Ҳ������������������������±������Ӧ�ó���ֻ��ʹ��`compile`���

```shell
[my-first-app] $ compile
```

![](consoleCompile.png)

##ִ�в���
�������������, ������������������Ҳ����ִ�в��ԡ�ֻ��ʹ��`test`���

```shell
[my-first-app] $ test
```

##��������ʽ����̨
����`console`�����뽻��ʽScala����̨, ������Խ���ʽ�ز������Ĵ���:

```shell
[my-first-app] $ console
```

Ҫ��scala����̨��������Ӧ��(����������ݿ�): `bash scala> new play.core.StaticApplication(new java.io.File("."))`

![](consoleEval.png)

##����
���������������̨ʱ����Play����һ��JPDA���Զ˿ڡ�Ȼ�������ʹ��Java debugger���ӵ��ԡ�����ʹ�������`activator -jvm-debug <port>`����:

```shell
$ activator -jvm-debug 9999
```

��JPDA�˿ڿ��ã�JVM����Ӧ������ʱ��ӡ������־��

```
Listening for transport dt_socket at address: 9999
```

##ʹ��sbt����
Play����̨����һ����ͨ��sbt����̨, ���������ʹ��sbt���ԣ���**triggered execution**.

����, ʹ�� `~ compile`:

```
[my-first-app] $ ~ compile
```

ÿ�������Դ�ļ�ʱ������ͻᱻ������

�����ʹ�� `~ run`:

```
[my-first-app] $ ~ run
```

������ģʽ����������ʱ����������������Ծͻᱻ���á�

ͬ����Ҳ������ `~ test`, ��ÿ�����޸�Դ�ļ�ʱ����ͣ���������Ŀ:

```
[my-first-app] $ ~ test
```

##ֱ��ʹ��play����
��Ҳ����ֱ������������������Play����̨������, ����`activator run`:

```
$ activator run
[info] Loading project definition from /Users/jroper/tmp/my-first-app/project
[info] Set current project to my-first-app (in build file:/Users/jroper/tmp/my-first-app/)

--- (Running the application from SBT, auto-reloading is enabled) ---

[info] play - Listening for HTTP on /0:0:0:0:0:0:0:0:9000

(Server started, use Ctrl+D to stop and go back to the console...)
```

Ӧ�ó����ֱ������������Ҫ�˳���������ʹ�� `Ctrl+D`, ��᷵�ص���ʾϵͳ�ն˵���ʾ�����档��Ȼ, **triggered execution**������Ҳ�ǿ��õ�:

```shell
$ activator ~run
```