# 文档搭建
## 安装

```shell
pip install mkdocs
```

## 创建项目

```shell
mkdocs new  .
```
- mkdocs.yml: 配置文件
- docs: 文档源文件

## 启动服务

```shell
mkdocs serve
```

## 添加文章

- 创建导航
```shell
nav:
    - 主页: 'index.md'
    - '云原生':
        - '基于kubeadm安装kubernetes集群': 'k8s/install_docker_k8s.md'
```

# 建立网站

```shell
 mkdocs build 
```

# 部署文档

```shell
mkdocs gh-deploy
```