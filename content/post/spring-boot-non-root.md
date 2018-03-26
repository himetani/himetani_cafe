+++
title = "非rootユーザでSpring Bootアプリをサービスとして起動させる"
date = "2017-06-11T21:26:03+09:00"
tags = ['spring-boot']
+++

非rootユーザでSpring Bootアプリをサービスとして起動させるためにやったことを書きました。

<!--more-->


# Spring Bootとは
- Springフレームワークをより手軽に使うことができるフレームワーク
- Embedded Tomcatを持ったexecutable jarがビルドで生成され、javaコマンドに渡すことでアプリを起動できる  


exucutable jarはとても魅力的な特徴で、~~Javaの開発とは思えないくらい~~簡単に、開発環境においてに開発、デバッグすることができます。
Javaの実行環境を用意さえすれば、簡単にアプリケーションを立ち上げることができます。
コンソール経由で実行する場合、特別な環境設定は不要です。(IDE経由で動かす場合は、Spring Boot系のプラグインの設定が必要)

exucatble jarを使ってアプリを起動するには以下のコマンドを使用します。

``` sh
# javaコマンドから起動
$ java -jar application.jar

# Spring Boot Maven plugin経由で起動
$ cd /path/to/pom.xml
$ mvn clean spring-boot:run
```

デフォルトの設定では、8080ポートでアプリケーションが起動されます。

# fully executable jar
前述した、javaコマンドやmvnコマンドを使ったアプリケーションの起動はフォアグラウンドプロセスで起動するため、そのままStaging環境やProduction環境のアプリケーションサーバにデプロイし、サービスとして起動することができません。
しかし、Spring Bootではサービスとして起動できるビルド成果物も簡単に作成することができます。
サービスとして起動できるビルド成果物のことを**fully executable jar**と言います。
pom.xmlに以下の設定を記述すると、ビルド後にtargetディレクトリ配下にできるjarファイルが**fully executable jar**となります。

``` xml 
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <executable>true</executable>
        <mode>service</mode>
    </configuration>
</plugin>
```

開発環境ではexecutable jarを、Staging環境やProduction環境ではfully executable jarをビルドしたいときは、Mavenのプロファイル機能を使ってビル成果物を切り替えることができます。

``` xml
<profiles>
    <profile>
        <id>dev</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <executable>false</etc.executable>
        </properties>
    </profile>
    <profile>
        <id>stg</id>
        <activation>
            <activeByDefault>false</activeByDefault>
        </activation>
        <properties>
            <executable>true</etc.executable>
        </properties>
    </profile>
    <profile>
        <id>prod</id>
        <activation>
            <activeByDefault>false</activeByDefault>
        </activation>
        <properties>
            <executable>true</etc.executable>
        </properties>
    </profile>
</profiles>
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <executable>${executable}</executable>
        <mode>service</mode>
    </configuration>
</plugin>
```

ビルドコマンド
``` sh
# dev
mvn clean package

# staging
mvn clean package -P stg

# production
mvn clean package -P prod
```


**fully executable jar**としてビルドされたjarファイルは、一般的なサービススクリプトのようにプロセスを制御することができます。init.dやsystemdに登録することもできます。

``` sh
$ cd /path/to/jar
$ ./application.jar [start | stop | status | restart]
```

# 非rootユーザで起動させる
実運用では、Spring Bootアプリのサービスを非rootユーザで起動させるようなケースがあると思います。
しかし、default設定では、非rootユーザで**fully executable jar**をサービスとして起動させることができません。
理由は二つあります。  
1. pidファイルの出力先ディレクトリに書き込み権限がない (defaultでは/var/run)  
2. logの出力先ディレクトリに書き込み権限がない       (defaultでは/var/log)

権限周りの設定変更がされていない限り、/var/runや/var/logには非rootユーザに書き込み権限はありません。
そこでpidファイルの出力先とlogファイルの出力先を変更する必要があります。

``` xml 
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <executable>true</executable>
        <mode>service</mode>
        <embeddedLaunchScriptProperties>
            <pidFolder>/path/to/pidFile</pidFolder>
            <logFolder>/path/to/logFile</logFolder>
        </embeddedLaunchScriptProperties>
    </configuration>
</plugin>
```

**fully executable jar**を起動させる非rootユーザに書き込み権限があるディレクトリを指定することで、サービスを起動できるようになります。

# まとめ
Spring Bootをサービスとして起動させるには、Spring Boot Maven Pluginの設定変更をする。  
1. ビルド成果物が**fully executable jar**となるように、<executable>オプションをtrueにする  
2. pidファイルとログの出力先を実行ユーザの書き込み権限のあるディレクトリに変更する(<pidFolder>, <logFolder>オプション)

# 参考
[59. Installing Spring Boot applications](https://docs.spring.io/spring-boot/docs/current/reference/html/deployment-install.html)
