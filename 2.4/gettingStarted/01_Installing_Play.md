#��װ Play

##׼������
����Ҫ�Ȱ�װ JDK 1.8 ����߰汾 (���� [һ�㰲װ����](#General-Installation-Tasks))��

##��������

* 1.�������°� [Typesafe Activator](https://typesafe.com/get-started)��
* 2.��ѹ����ѹ��������дȨ�޵�λ�á�
* 3.���ն����� `cd activator*` �л������Ŀ¼ (��ʹ���ļ�������)
* 4.���ն����� `activator ui` ����(��ʹ���ļ�������)
* 5.���� [http://localhost:8888](http://localhost:8888/)

��ᷢ�ֺܶ�Ӧ�ó����б���ĵ�������������ٿ�ʼ�������ȼ���һ�� play-java ���塣

###������
Ҫ��������ļ�ϵͳ���κεط�������ֱ��ʹ�� play ����, ��Ҫ��activator����Ŀ¼��ӵ����ϵͳ������Path�� (���� [һ�㰲װ����](#General-Installation-Tasks)).

����һ������`play-java`��ģ��`my-first-app`������ô�򵥣�

```
activator new my-first-app play-java
cd my-first-app
activator run
```

Ȼ����� http://localhost:9000��

�������Ѿ�׼����ʹ��Play��!

##<a name="General-Installation-Tasks"></a>һ�㰲װ����
�������Ҫ����һЩһ�������Ա㰲װPlay!

###JDK ��װ
��ȷ�����ϵͳ���� JDK (Java Development Kit) 1.8 ����߰汾��ʹ������������������֤��

```
java -version
javac -version
```

����㻹û�а�װ JDK, ����Ҫ��װ����

* 1.MacOSϵͳ��Java �����õ�, ���������Ҫ[���������°汾](http://www.oracle.com/technetwork/java/javase/downloads/index.html)
* 2.Linuxϵͳ�� ʹ�����µ�Oracle JDK��OpenJDK(��Ҫʹ�÷�gcj).
* 3.Windowsϵͳ��ֻ��Ҫ���غͰ�װ[���°��JDK��װ��](http://www.oracle.com/technetwork/java/javase/downloads/index.html)��

###����ִ���ļ���ӵ�����������Path��
Ϊ�����������Ӧ����� Activator ��װĿ¼��ϵͳ����������`PATH`�С�

��Unix��ϵͳ, ʹ�� `export PATH=/path/to/activator:$PATH`

��Windowsϵͳ, ��� `;C:\path\to\activator` ����Ļ��������� `PATH` �����С�·���в�Ҫʹ�ÿո�

###�ļ�Ȩ��
####Unix
�ڷ����������� activator��д��һЩ�ļ���Ŀ¼, ���Բ�Ҫ��װ�� /opt, /usr/local ���κ���������Ҫ���ⶨ��Ȩ�޵�λ�á�

ȷ�� activator �ű��ǿ�ִ�еġ��������, �����ն����� `chmod u+x /path/to/activator` �����Ȩ�ޡ�

###��������
�����������һ��������棬ȷ��������������ã�
��Windows����`set HTTP_PROXY=http://<host>:<port>` ��
������UNIX��ϵͳ��ʹ�� `export HTTP_PROXY=http://<host>:<port>` ��