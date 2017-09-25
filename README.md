# JCConf 2017 - Maven 上的那些 Jar 是如何鍊成的

Maven 真的很方便, 你是否曾經也像我一樣, 好奇要怎麼發佈自己的 library 到 maven, 讓別人也很方便的使用? 這次將以前的一些經驗整理成懶人包, 用最短的時間內跟大家分享。

> 這邊會有一張懶人包圖

這次的懶人包, 是以 [GitHub](https://github.com) 作為 SCM, 並整合 [Travis CI](https://travis-ci.org) 來幫我們自動測試及部署, 目的是讓 Travis CI 自動偵測:

- 每次 push 或 pull request 到 GitHub 時, 自動執行 test
- 每次建立 tag 或 draft a new release 時, 自動發佈到 Maven central repository

## OSSRH Setup

Sonatype OSSRH (OSS Repository Hosting) 上管理了這些 open source, 因此你必須也要註冊一個帳號來上傳自己的 artifacts

1. 註冊 [JIRA 帳號](https://issues.sonatype.org/secure/Signup!default.jspa)
2. 申請 [*groupId* 的使用權限](https://issues.sonatype.org/secure/CreateIssue.jspa?issuetype=21&pid=10134)

以 GitHub 作為 SCM 的話, [Maven 建議](http://central.sonatype.org/pages/choosing-your-coordinates.html) *groupId* 為: *com.github.yourusername*

## Configure Maven build

### POM

Maven 對於 [pom.xml 的結構](http://central.sonatype.org/pages/requirements.html#sufficient-metadata)有些基本要求, 我將實際上在使用的簡化後, 提供了一個完整的 [pom.xml 範例](https://github.com/shihyuho/jcconf2017-uploading-maven-artifacts/blob/master/pom.xml) (基本上複製貼上後再稍作修改即可)

### Source & Java Doc

Maven 也會要求上傳時同時發佈 source 以及 javadoc (就算你沒有也要產生這些檔案), 因此我們可以透過 *maven-javadoc-plugin* 及 *maven-source-plugin* 這兩個 plugin 來協助我們產生, 範例也同樣放在 [pom.xml 範例](https://github.com/shihyuho/jcconf2017-uploading-maven-artifacts/blob/master/pom.xml) 

設定好後可以執行:

```
$ mvn clean verify
```

在 `target/` 下就會產生 `{artifactId}-{version}-sources.jar` 以及 `{artifactId}-{version}-javadoc.jar` 兩個檔案



## GPG Signatures

任何發佈的 artifacts 都使用 GPG 簽章

### Install

macos 可以透過 [Homebrew](https://brew.sh) 安裝:

```
$ brew install gnupg2
```

其他系統可從 [https://www.gnupg.org/download/](https://www.gnupg.org/download/) 下載安裝

### Generating a Key Pair

建立新的 Key, passphrase 必須要好好保存, 後面整合 travis 時會用到

```
$ gpg --gen-key
```

查看 public key

```
$ gpg --list-keys
/Users/Matt/.gnupg/pubring.gpg
------------------------------
pub   2048R/{key-id} 2016-12-15
...
```

這邊可以看到目前有一把 public key, 接著要把這把 key 發佈到 *sks-keyservers.net*

```
$ gpg --keyserver hkp://pool.sks-keyservers.net --send-keys {key-id}
```

查看 Secret key

```
$ gpg --list-secret-keys
/Users/Matt/.gnupg/secring.gpg
------------------------------
sec   2048R/{key-id} 2016-12-15
...
```

將 Secret key 匯出來, 後面整合 travis 時會用到

```
gpg -a --export-secret-key {key-id} > .travis.key.gpg
```

## Travis CI Integration

這次選擇了整合 [Travis CI](https://travis-ci.org) 來幫我們自動測試及部署, 請先註冊好帳號

### Travis CLI

[Travis CLI](https://github.com/travis-ci/travis.rb#installation) 可以幫我們把重要資訊加密, 安裝前請確保系統上已經先安裝好 [Ruby](http://www.ruby-lang.org/en/downloads/), macos 可以下

```
$ brew install ruby
```

接著安裝 CLI:

```
$ gem install travis -v 1.8.8 --no-rdoc --no-ri

$ travis version
1.8.8

$ travis login
# 試著登入 travis
```

### Setup the Travis build

Repository 的 root 要多加入 3 個檔案, 結構會類似:

```
.
├── .travis.yml // 這是 Travis CI 規定的檔名
├── .travis.key.gpg // 這是這次範例取的檔名, 可以做修改
├── .travis.settings.xml // 這是這次範例取的檔名, 可以做修改
├── README.md
├── pom.xml
└── src
```

#### .travis.yml

`.travis.yml` 是 Travis CI 的設定檔案：

```yaml
sudo: false
language: java
jdk:
- oraclejdk8
addons:
  apt:
    packages:
    - oracle-java8-installer
env:
  global:
  - secure: ...
  - secure: ...
  - secure: ...
script: travis_wait 360 mvn clean test -U
before_deploy:
  - gpg --import .travis.key.gpg
  - mvn versions:set -DnewVersion=${TRAVIS_TAG}
deploy:
  provider: script
  skip_cleanup: true
  script: mvn clean deploy -P release --settings .travis.settings.xml
  on:
    tags: true
```

其中 `env.global` 下有三個 secure 變數, 我們會透過 Travis CLI 來自動產生:

```
$ travis encrypt -r {travis-username}/{repository} "OSSRH_USERNAME={your-ossrh-username}" --add
$ travis encrypt -r {travis-username}/{repository} "OSSRH_PASSWORD={your-ossrh-password}" --add
$ travis encrypt -r {travis-username}/{repository} "GPG_PASSPHRASE={your-gpg-passphrase}" --add
```

#### .travis.key.gpg

這就是匯出的 [GPG private key](#generating-a-key-pair), 不用擔心會被使用, 因為它已經被 passphrase 加過密, 而 passphrase 又被再加了一層密了

#### .travis.settings.xml 

這是讓 Travis CI 讀取的 maven 設定檔

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                      http://maven.apache.org/xsd/settings-1.0.0.xsd">
    <servers>
        <server>
            <id>ossrh</id>
            <username>${env.OSSRH_USERNAME}</username>
            <password>${env.OSSRH_PASSWORD}</password>
        </server>
    </servers>
    <profiles>
        <profile>
            <id>release</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <properties>
                <gpg.executable>gpg</gpg.executable>
                <gpg.keyname>{your-private-key-id}</gpg.keyname>
                <gpg.passphrase>${env.GPG_PASSPHRASE}</gpg.passphrase>
            </properties>
        </profile>
    </profiles>
</settings>
```

### Maven Central Repository

透過以上的設定就算是完成了, 發佈後你可以用瀏覽器到 [https://oss.sonatype.org](https://oss.sonatype.org) 瀏覽檢查上傳結果, 或執行:

```
$ curl http://search.maven.org/solrsearch/select?q=g:"{your-groupId}"&wt=json | jq .
```

> 使用 [jq](https://stedolan.github.io/jq/) 來對 format json, 需要另外安裝, 或執行指令時最後不下 `| jq .`

最後當然 repository 的 `README.md` 上也要記得加上 Travis CI 測試結果的圖案: 

```
[![Build Status](https://travis-ci.org/{travis-username}/{repository}.svg?branch=master)](https://travis-ci.org/{travis-username}/{repository})
```

## Reference

- [Choosing your Coordinates](http://central.sonatype.org/pages/choosing-your-coordinates.html)
- [Guide to uploading artifacts to the Central Repository](https://maven.apache.org/guides/mini/guide-central-repository-upload.html)
- [Working with PGP Signatures](http://central.sonatype.org/pages/working-with-pgp-signatures.html)
- [OSSRH Guide](http://central.sonatype.org/pages/ossrh-guide.html)
- [Deploying to OSSRH with Apache Maven](http://central.sonatype.org/pages/apache-maven.html)
- [Maven Search Engine API](http://search.maven.org/#api)
