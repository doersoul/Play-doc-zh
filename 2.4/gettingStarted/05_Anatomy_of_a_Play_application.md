#����һ��PlayӦ�ó���

##PlayӦ�ó��򲼾�
PlayӦ�ó���Ĳ����Ǳ�׼���ģ��������龡���ܼ򵥡����״γɹ�����֮��,һ��PlayӦ�ó�������������:

```
app                      �� Ӧ�ó�����ҪԴ����
 �� assets                �� ������ʲ�Դ�ļ�
    �� stylesheets        �� ͨ���� LESS CSS Դ����
    �� javascripts        �� ͨ���� CoffeeScript Դ����
 �� controllers           �� Ӧ�ó��������
 �� models                �� Ӧ�ó�����ҵ�߼���
 �� views                 �� ģ��
build.sbt                �� Ӧ�ó��򹹽��ű�
conf                     �� �����ļ��������Ǳ�����Դ (��classpath)
 �� application.conf      �� �������ļ�
 �� routes                �� ·�ɶ���
dist                     �� ��������Ĺ��̷����е������ļ�
public                   �� �����ʲ�
 �� stylesheets           �� CSS �ļ�
 �� javascripts           �� Javascript �ļ�
 �� images                �� ͼ���ļ�
project                  �� sbt �����ļ�
 �� build.properties      �� sbt ��Ŀ�ı��
 �� plugins.sbt           �� sbt �������Play ��������
lib                      �� ���йܵĿ�������
logs                     �� ��־�ļ���
 �� application.log       �� Ĭ����־�ļ�
target                   �� ����������
 �� resolution-cache      �� ������������Ϣ
 �� scala-2.10
    �� api                �� ���ɵ� API docs
    �� classes            �� ��������ļ�
    �� routes             �� ��·�����ɵ�Դ
    �� twirl              �� ��ģ�����ɵ�Դ
 �� universal             �� Ӧ�ó�����
 �� web                   �� ����� web �ʲ�
test                     �� ��Ԫ�͹��ܲ��Ե�Դ�ļ���
```

##`app/` Ŀ¼
`app`Ŀ¼�������п�ִ�е�artifacts: Java��ScalaԴ����, ģ��ͱ�����ʲ���Դ�ļ���

��`app`Ŀ¼��������, ��MVC�ܹ�ģʽ��ÿһ�����:

* `app/controllers`
* `app/models`
* `app/views`

��Ȼ���������Լ��İ�, ����һ�� `app/utils` ����

> ע����Play, controllers, models �� views ������������ֻ��һ��Լ��������Ҫ���Ը�����(����ÿ��������`com.yourcompany`Ϊǰ׺)��

���ﻹ��һ����ѡ��Ŀ¼�� `app/assets`��Ϊ�˱���ĳЩ�ʲ�����[LESSԴ�ļ�](http://lesscss.org/)��[CoffeeScriptԴ�ļ�](http://coffeescript.org/)��

##`public/` Ŀ¼
��Դ�洢��`public`Ŀ¼������һЩ��̬���ʲ�����Щ��Web������ֱ���ṩ����

���Ŀ¼�ֳ�������Ŀ¼������ͼ��, CSS��ʽ���JavaScript�ļ�����Ӧ��������������ľ�̬�ʲ����Ա�������PlayӦ�ó����һ���ԡ�

> ��һ���½���Ӧ�ó���,  `/public` Ŀ¼ӳ�䵽 `/assets` URL·��, ������Ժ����׵ĸ�����, ����������Ϊ��ľ�̬�ʲ�ʹ�ü���Ŀ¼��

##`conf/` Ŀ¼
`conf` Ŀ¼����Ӧ�ó���������ļ��������ж�����Ҫ�������ļ�:

* `application.conf`, Ӧ�ó�����������ļ�, �������ò���
* `routes`, ·�ɶ����ļ�

�������Ҫ����ض���Ӧ�ó��������ѡ��, ��Ӹ���ѡ�`application.conf`�ļ���һ�������⡣

���һ������Ҫһ���ض��������ļ�, ����������`conf`Ŀ¼�µ��ļ���

##`lib/` Ŀ¼
`lib` Ŀ¼�ǿ�ѡ�ģ��������йܿ�������ϵ, ������������Ҫ�ڹ���ϵͳ���ֶ������JAR�ļ���ֻ��Ҫ�϶��κ�JAR�ļ���������ǻ���ӵ����Ӧ�ó����Classpath��

##`build.sbt` �ļ�
�����Ŀ������������ͨ�����ڹ��̸�Ŀ¼�µ�`build.sbt`�С��� `project/` Ŀ¼�е� `.scala`�ļ�Ҳ���������������Ŀ�Ĺ�����

##`project/` Ŀ¼
`project` Ŀ¼����sbt��������:

* `plugins.sbt` ���������Ŀʹ�õ�sbt���
* `build.properties` ���������������app��sbt�汾��

##`target/` Ŀ¼
`target` Ŀ¼��������ϵͳ���ɵ����ж����������˽�����������ʲô�����Ǻ����õġ�

* `classes/` �������б������(��JavaԴ�����ScalaԴ����).
* `classes_managed/` ������ͨ����ܹ������(��ͨ��·�ɻ�ģ��ϵͳ���ɵ���)�����������IDE��Ŀ�У���Ӵ����ļ�����Ϊ�ⲿ���ļ��к����á�
* `resource_managed/` �������ɵ���Դ, ͨ���Ǳ�����ʲ�����LESS CSS��CoffeeScript����Ľ����
* `src_managed/` �������ɵ�Դ�ļ�, ��ͨ��ģ��ϵͳ���ɵ�ScalaԴ�ļ���
* `web/` ����ͨ��sbt-web������ʲ�������Щ`app/assets` �� `public` �ļ����еġ�

##���͵� .gitignore �ļ�
ĳЩ���ɵ��ļ���Ӧ�ñ���İ汾����ϵͳ���ԡ�������һ�����PlayӦ�õ��͵� `.gitignore` �ļ�:

```
logs
project/project
project/target
target
tmp
dist
.cache
```

##Ĭ�� SBT ����
��Ҳ����ѡ����SBT��Maven��Ĭ�ϲ��֡���ע�����������ʵ���ԵĺͿ��ܻ���Щ���⡣Ϊ��ʹ���������, ��ʹ�� `disablePlugins(PlayLayoutPlugin)`��������ֹͣPlay��дĬ��SBT����, ������������:

```
build.sbt                  �� Ӧ�ó��򹹽��ű�
src                        �� Ӧ�ó���Դ����
 �� main                    �� ������ʲ�Դ
    �� java                 �� Java Դ
       �� controllers       �� Java ������
       �� models            �� Java ��ҵ�߼���
    �� scala                �� Scala Դ
       �� controllers       �� Scala ������
       �� models            �� Scala ��ҵ�߼���
    �� resources            �� �����ļ��������Ǳ�����Դ (�� classpath)
       �� application.conf  �� �������ļ�
       �� routes            �� ·�ɶ���
    �� twirl                �� ģ��
    �� assets               �� ������ʲ�Դ
       �� css               �� ͨ����LESS CSS Դ�ļ�
       �� js                �� ͨ����CoffeeScript Դ�ļ�
    �� public               �� �����ʲ�
       �� css               �� CSS �ļ�
       �� js                �� Javascript �ļ�
       �� images            �� ͼ���ļ�
 �� test                    �� ��Ԫ���ܲ���
    �� java                 �� ��Ԫ���ܲ��Ե�Java Դ�ļ���
    �� scala                �� ��Ԫ���ܲ��Ե�Scala Դ�ļ���
    �� resources            �� ��Ԫ���ܲ��Ե���Դ�ļ���
 �� universal               �� �����������Ŀ�����е������ļ�
project                    �� sbt �����ļ�
 �� build.properties        �� sbt ��Ŀ�ı��
 �� plugins.sbt             �� sbt �������Play ��������
lib                        �� ���йܵĿ�����
logs                       �� ��־�ļ���
 �� application.log         �� Ĭ����־�ļ�
target                     �� ���ɵĶ���
 �� scala-2.10.0            
    �� cache              
    �� classes              �� ��������ļ�
    �� classes_managed      �� ��������ļ� (ģ��, ...)
    �� resource_managed     �� �������Դ (less, ...)
    �� src_managed          �� ���ɵ�Դ (ģ��, ...)
```