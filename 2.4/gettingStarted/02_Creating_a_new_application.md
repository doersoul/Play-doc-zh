#����һ����Ӧ�ó���

##��activator�����һ����Ӧ�ó���
`activator` �������������һ����PlayӦ�ó���Activator������ѡ��һ��ģ�壬���Ӧ�ó��򽫻������ģ�崴����ƽ����Play��Ŀ����Щ������`play-scala` ��ģ��������������Scala��Play��Ӧ�ó���ͬʱ������`play-java`��������������Java��PlayӦ�ó���

> ��ע�⣬��ѡ��Scala��Javaģ����һ���ϲ�����ζ�����Ժ��ܸ������ԡ�����, ֻҪ��ϲ�����㶼����ʹ��Ĭ�ϵ�JavaӦ�ó���ģ�崴��һ����Ӧ�ó���Ȼ��ʼ���Scala���롣

Ҫ����һ����ͨ��Play Scala Ӧ�ó�������:

```shell
$ activator new my-first-app play-scala
```

Ҫ����һ����ͨ��Play Java Ӧ�ó�������:

```shell
$ activator new my-first-app play-java
```

��֮�������������Ҫ�������滻��� `my-first-app` ��Activator ��ʹ�����������ΪĿ¼�����������洴�����������Ҳ�������Ժ����������֡�

![](activatorNew.png)

> �����ϣ��ʹ������Activatorģ��, ���������`activator new`���������ʾ������һ��Ӧ�ó�����, Ȼ�����������ѡ��һ�����ʵ�ģ�塣

һ��Ӧ�ó��򱻴�������Ϳ����ٴ�ʹ��`activator`�������[Play����̨](03_Using_the_Play_console.md)��

```shell
$ cd my-first-app
$ activator
```

##��Activator UI����һ����Ӧ�ó���
Ҳ����ʹ��Activator UI����һ����PlayӦ�ó���Ҫʹ�� Activator UI, ���У�

```shell
$ activator ui
```

������Ķ�[����](https://typesafe.com/activator/docs)���ĵ��˽����ʹ��Activator UI��

##����ActivatorҲ���Դ���һ����Ӧ�ó���
���ð�װActivatorҲ���Դ���һ����PlayӦ�ó���, ֱ��ʹ��`sbt`��

> ��������Ȱ�װ[sbt](http://www.scala-sbt.org/)��

1.Ϊ���Ӧ�ó��򴴽�һ����Ŀ¼��Ȼ���������sbt�����ű��ͼ��϶����������ݣ���`project/plugins.sbt`��,����:

```
// The Typesafe repository
resolvers += "Typesafe repository" at "https://repo.typesafe.com/typesafe/releases/"

// Use the Play sbt plugin for Play projects
addSbtPlugin("com.typesafe.play" % "sbt-plugin" % "2.4.x")
```

��ظ�������� 2.4.x ������Ҫ�õİ汾���������Ҫʹ�ÿ��հ汾, ����Ҫ����ָ����Щ������:

```
// Typesafe snapshots
resolvers += "Typesafe Snapshots" at "https://repo.typesafe.com/typesafe/snapshots/"
```

2.Ҫʹ����ȷ��sbt�汾, ȷ����`project/build.properties`������������:

```scala
sbt.version=0.13.8
```

3.����������`build.sbt`�ļ���

��Java��Ŀ���м���:

```scala
name := "my-first-app"

version := "1.0"

lazy val root = (project in file(".")).enablePlugins(PlayJava)
```

������ Scala ��Ŀ�м���:

```
name := "my-first-app"

version := "1.0.0-SNAPSHOT"

lazy val root = (project in file(".")).enablePlugins(PlayScala)
```

Ȼ�������Ŀ¼���� sbt����̨:

```shell
$ cd my-first-app
$ sbt
```

sbt���Զ���ȡ��Ŀ��Ҫ�������