#���������ѡIDE

ʹ��Play�����ס�����������Ҫ���ӵ�IDE, ��ΪPlay�����޸�Դ�ļ��ǻ��Զ������ˢ�£�������������ɵ�ʹ��һ���򵥵��ı��༭����

����ʹ��һ���ִ�Java��Scala IDE�����ṩǿ��Ĺ��ܣ����Զ����,��ʱ����,�ع������͵��ԡ�

##Eclipse
###���� sbteclipse
Play��Ҫ[sbteclipse](https://github.com/typesafehub/sbteclipse) 4.0.0 ����°汾������������ݵ���� project/plugins.sbt�ļ�:

```
addSbtPlugin("com.typesafe.sbteclipse" % "sbteclipse-plugin" % "4.0.0")
```

������`eclipse`����֮ǰ���������`����`��Ĺ��̡�ͨ����`build.sbt` ����������ã������ǿ�Ƶ�`eclipse`��������ʱ�Զ����б���:

```scala
// ����Eclipse�ļ�ǰ������Ŀ, ������ɵ� .scala �� .class �ļ�Ϊ��ǰ views �� routes
EclipseKeys.preTasks := Seq(compile in Compile)
```

��������Ŀ����ScalaԴ����, ����Ҫ��װ[Scala IDE](http://scala-ide.org/)��

����㲻�밲װScala IDE�������Ŀ�н���JavaԴ����, ��ô����԰���������:

```scala
EclipseKeys.projectFlavor := EclipseProjectFlavor.Java           // Java ��Ŀ. ����Scala IDE
EclipseKeys.createSrc := EclipseCreateSrc.ValueSet(EclipseCreateSrc.ManagedClasses, EclipseCreateSrc.ManagedResources)  // Ϊ��ͼ��·��ʹ�� .class �ļ��������� .scala �ļ��� 
```

###��������
Play�ṩһ�������Լ�[Eclipse](https://eclipse.org/)���á�Ҫת��PlayӦ�ó��򵽿��Թ�����Eclipse����, ʹ��`eclipse`���

```shell
[my-first-app] $ eclipse
```

�������ץȡ���õ�jarsԴ (��Ứ�Ѻܳ�ʱ�䲢��һЩԴ���ܻᶪʧ):

```shell
[my-first-app] $ eclipse with-source=true
```

> ע�������ʹ�ü�������Ŀ, ����Ҫ��`build.sbt`���ʵ�����`skipParents`:

```scala
EclipseKeys.skipParents in ThisBuild := false
```

���play����̨, ����:

```scala
[my-first-app] $ eclipse skip-parents=false
```

Ȼ������Ҫ�� **File/Import/General/Existing project��** �˵���Ӧ�ó����뵽��Ĺ����ռ�(��Ҫ�ȱ��������Ŀ)��

![](eclipse.png)

Ҫ����, �� `activator -jvm-debug 9999 run` �������Ӧ�ó��򣬲���Eclipse���һ���Ŀ��ѡ�� **Debug As, Debug Configurations**���� **Debug Configurations** ����, �һ� **Remote Java Application** ��ѡ�� **New**�����˿ڸ�Ϊ9999�͵�� **Apply**��������������Ե�� **Debug** �����ӵ��������е�Ӧ�ó���ֹͣ���ԻỰ����ֹͣ��������

> ��ʾ: �����ʹ�� `~run` ���������Ӧ�ó��������ø����ļ�ʱ�Զ�ֱ�ӱ���Ĺ��ܡ�����������`��ͼ`Ŀ¼����һ����ģ��ʱ��scalaģ���ļ��ᱻ�Զ����֣����ҵ��ļ�����ʱ�Զ����롣�����ʹ����ͨ��`run`����ôÿ���㶼Ҫ�ֶ�����������`ˢ��`��

�������Ӧ�ó�����������Ҫ�ĸ���, ��ı�classpath, Ҫ�ٴ�ʹ��`eclipse`���������������ļ���

> ��ʾ: ������һ���Ŷ��й���ʱ����Ҫ�ύEclipse�����ļ�!

���ɵ������ļ���������Ŀ�ܰ�װλ�õľ������á���Щ�ض������Լ��İ�װλ�á����㹤����һ���Ŷ���ʱ, ÿ�������߱��뱣������Eclipse�����ļ���˽�еġ�


##IntelliJ
[Intellij IDEA](https://www.jetbrains.com/idea/)��������ʹ��������ʾ�������Կ��ٴ���һ��PlayӦ�ó����㲻��Ҫ����IDE������κζ���, SBT�������߻ᰲ�������ʵ��Ŀ�, ���������͹�����Ŀ��

��IntelliJ IDEA�п�ʼ����һ��PlayӦ�ó���֮ǰ, ��ȷ���Ѱ�װ����Scala���������������ʹ�㲻��Scala����, ��Ҳ�������ʹ��ģ������ͽ�������

Ҫ����һ��PlayӦ�ó���:

1. �� **New Project** ��, ѡ���� **Scala** ����� **Play 2.x** ����� **Next**��
2. ���������Ŀ��Ϣ����� **Finish**.

IntelliJ IDEA ��ʹ��SBT����һ����Ӧ�ó���

��Ҳ���Ե���һ���Ѵ��ڵ�Play��Ŀ��

Ҫ����һ��Play��Ŀ:

1. �� Project wizard, ѡ�� **Import Project**��
2. �ڴ򿪵Ĵ���, ѡ�����뵼�����Ŀ������� **OK**��
3. ���򵼵���һҳ, ѡ�� **Import project from external model** ѡ��, ѡ�� **SBT project** �͵�� **Next**��
4. ���򵼵���һҳ��ѡ�񸽼ӵ���ѡ������ **Finish**��

�����Ŀ�ṹ, ȷ�����б���������������ء������ʹ�ô��븨��, �����Ͷ�̬����������ܡ�

��������иմ�����Ӧ�ó��򣬲������������Ĭ��λ��`http://localhost:9000`�鿴�����Ҫ����PlayӦ�ó���:

1. ����һ���µ�Run���� �C �����˵�, ѡ�� Run -> Edit Configurations
2. ��� + �����һ���µ�����
3. �������б�ѡ�� ��SBT Task��
4. �� ��tasks�� �����, ������ ��run��
5. Ӧ�ø��ĺ�ѡ�� OK
6. ��������Դ����˵���Run�˵���ѡ�� ��Run�����������Ӧ�ó���

���������ΪPlayӦ�ó���ʼһ�����ԻỰ��ʹ��Ĭ�� Run/Debug �����趨��

Ҫ�˽����ϸ��Ϣ, ���������Play Framework 2.x �̳�:

[https://confluence.jetbrains.com/display/IntelliJIDEA/Play+Framework+2.0](https://confluence.jetbrains.com/display/IntelliJIDEA/Play+Framework+2.0)

###�Ӵ���ҳ�浼����Դ����λ��
ʹ�� `play.editor` ����ѡ��, ���������Play��ӳ����ӵ�һ������ҳ�档���, ����ԴӴ���ҳ�����ɵ�����IntelliJ, ֱ�ӽ���Դ����λ��(����Ҫ���Ȱ�װ Remote Call [https://github.com/Zolotov/RemoteCall](https://github.com/Zolotov/RemoteCall) IntelliJ ���)��

ֻҪ��װRemote Call ��������������ѡ���������Ӧ�ó���:

```
-Dplay.editor=http://localhost:8091/?message=%s:%s -Dapplication.mode=dev
```


##Netbeans
###��������
ĿǰPlayû��ԭ����Netbeans��Ŀ����֧��, ����һ��NetBeans��Scala������԰���ʹ��Scala���Ժ�SBT:

[https://github.com/dcaoyuan/nbscala](https://github.com/dcaoyuan/nbscala)

����һ��SBT����ɴ���Netbeans��Ŀ����:

[https://github.com/dcaoyuan/nbsbt](https://github.com/dcaoyuan/nbsbt)

##ENSIME
###��װ ENSIME
���հ�װ˵������ [https://github.com/ensime/ensime-emacs](https://github.com/ensime/ensime-emacs)��

###��������
�༭ project/plugins.sbt �ļ�, ����������� (��Ӧ���ȼ�� [https://github.com/ensime/ensime-sbt](https://github.com/ensime/ensime-sbt) ��ò�����°汾):

```
addSbtPlugin("org.ensime" % "ensime-sbt" % "0.1.5-SNAPSHOT")
```

���� Play:

```
$ activator
```

��play����̨���� ��ensime generate�������������һ�� .ensime �ļ���play��Ŀ�ĸ�Ŀ¼�¡�

```
$ [MYPROJECT] ensime generate
[info] Gathering project information...
[info] Processing project: ProjectRef(file:/Users/aemon/projects/www/MYPROJECT/,MYPROJECT)...
[info]  Reading setting: name...
[info]  Reading setting: organization...
[info]  Reading setting: version...
[info]  Reading setting: scala-version...
[info]  Reading setting: module-name...
[info]  Evaluating task: project-dependencies...
[info]  Evaluating task: unmanaged-classpath...
[info]  Evaluating task: managed-classpath...
[info] Updating {file:/Users/aemon/projects/www/MYPROJECT/}MYPROJECT...
[info] Done updating.
[info]  Evaluating task: internal-dependency-classpath...
[info]  Evaluating task: unmanaged-classpath...
[info]  Evaluating task: managed-classpath...
[info]  Evaluating task: internal-dependency-classpath...
[info] Compiling 5 Scala sources and 1 Java source to /Users/aemon/projects/www/MYPROJECT/target/scala-2.9.1/classes...
[info]  Evaluating task: exported-products...
[info]  Evaluating task: unmanaged-classpath...
[info]  Evaluating task: managed-classpath...
[info]  Evaluating task: internal-dependency-classpath...
[info]  Evaluating task: exported-products...
[info]  Reading setting: source-directories...
[info]  Reading setting: source-directories...
[info]  Reading setting: class-directory...
[info]  Reading setting: class-directory...
[info]  Reading setting: ensime-config...
[info] Wrote configuration to .ensime
```

###���� ENSIME
��Emacs, ִ�� M-x ensime �������Ļ�ϵ�˵����

���������������Play��ĿӦ���������ͼ��, �Զ���ɣ��ȵȡ�ע��, ���������µĿ����������play��Ŀ,����Ҫ�������� ��ensime generate�� ������ENSIME��

###������Ϣ
��� ENSIME README�� �� [https://github.com/ensime/ensime-emacs](https://github.com/ensime/ensime-emacs)���������ʲô����, ������ϵ���ǵ�ensime�û��飬 �� [https://groups.google.com/forum/?fromgroups=#!forum/ensime](https://groups.google.com/forum/?fromgroups=#!forum/ensime)��

##������Ҫ�� Scala���
Scala��һ���Ƚ��µı������, ���Թ������ɲ���ṩ��������IDE���ġ�

1. Eclipse Scala IDE: [http://scala-ide.org/](http://scala-ide.org/)
2. NetBeans Scala���: [https://github.com/dcaoyuan/nbscala](https://github.com/dcaoyuan/nbscala)
3. IntelliJ IDEA Scala���: [http://confluence.jetbrains.net/display/SCA/Scala+Plugin+for+IntelliJ+IDEA](http://confluence.jetbrains.net/display/SCA/Scala+Plugin+for+IntelliJ+IDEA)
4. IntelliJ IDEA�Ĳ�����ڻ�����չ, ����ʹ��nightly�������ܻ���㸽�ӹ��ܣ�ֻ�Ƕ໨һ��ɱ���
5. Nika (11.x) ����ֿ�: [https://www.jetbrains.com/idea/plugins/scala-nightly-nika.xml](https://www.jetbrains.com/idea/plugins/scala-nightly-nika.xml)
6. Leda (12.x) ����ֿ�: [https://www.jetbrains.com/idea/plugins/scala-nightly-leda.xml](https://www.jetbrains.com/idea/plugins/scala-nightly-leda.xml)
7. IntelliJ IDEA Play ���(����Leda 12.x����): [http://plugins.intellij.net/plugin/?idea&pluginId=7080](http://plugins.intellij.net/plugin/?idea&pluginId=7080)
8. ENSIME - Emacs��Scala IDEģʽ: [https://github.com/aemoncannon/ensime](https://github.com/aemoncannon/ensime)(��������� ENSIME/Playʹ��˵��)