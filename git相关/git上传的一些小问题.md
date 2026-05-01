# 一、git 忽略某个目录或文件不上传

解决方法：https://blog.csdn.net/sunxiaoju/article/details/86495234

# 二、大于 100MB 文件上传的解决办法

## 1.解决办法：https://git-lfs.com/

## 2.Git LFS 锁定 API 不支持

- 报错：`Remote "origin" does not support the Git LFS locking API.`
- 解决办法：
  git config lfs.git 项目网址 false
- 该解决办法的问题：这可能影响协作时的文件锁定机制
