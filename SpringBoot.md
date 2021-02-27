将常用的框架组合起来，提供默认的配置，提供可拔插的设计，即starter

简化了Spring繁重的配置，可以避免依赖版本冲突



**常用注解**

@SpringBootApplication：Spring Boot核心注解，包含以下注解

​	@SpringBootConfiguration:组合了@Configuration注解，实现配置文件的功能

​	@EnableAutoConfiguration:打开自动配置功能

​	@ComponentScan:Spring组件扫描

SpringBoot将常见的开发功能，包装成场景启动器(starter)



配置加载顺序

1.properties文件

2.yaml文件

3.系统环境白能量

4.命令行参数



bootstrap.yaml与application.yaml

先加载bootstrap，在ApplicationContext的引导阶段生效，用于读取配置中心数据，spring clould config或nacos中用到





