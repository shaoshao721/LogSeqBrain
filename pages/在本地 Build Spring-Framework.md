- 前置步骤
	- 安装 Java8 及以上环境
	- 从官方 github 仓库克隆源码
	- 切换到想要编译的 tag 下
	- 阅读官方 Build from Source 的文档
- 步骤
	- Build
		- ``` bash
		  ./gradlew build
		  ```
	- Test
		- ``` bash
		  ./gradlew -a :spring-webmvc:test
		  ```
	- Install to local maven repo
		- ``` bash
		  ./gradlew publishToMavenLocal -x api -x javadoc -x dokkaHtmlMultiModule -x asciidoctor -x asciidoctorPdf -x distZip
		  ```
	-