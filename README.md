blog by stephen.sun
# 恢复博客
1. 安装git、nodejs
2. git clone git@github.com:gameloses/gameloses.github.io.git
3. git checkout -b dev origin/dev #checout远程dev分支创建本地dev分支
4. git checkout dev
5. git submodule init
6. git submodule update
7. 在文件夹内执行命令:
```
  npm install hexo-cli -g
  npm install
  npm install hexo-deployer-git --save
```

# 博客修改
`hexo g -d ` #生成静态网页部署到github上
# 博客备份
```
git add .
git commit -m "提交文件"
git push origin dev
```
# 附录
   Hexo的源文件说明
1. _config.yml站点的配置文件
2. themes/主题文件夹
3. source博客文章的.md文件
4. scaffolds/文章的模板
5. package.json安装包的名称
6. .gitignore限定在push时哪些文件可以忽略
7. .git/主题和站点都有，标志这是一个git项目
8. node_modules/是安装包的目录,npm install重新生成，不需要拷贝
9. public是hexo g生成的静态网页，不需要拷贝
10. .deploy_git同上，hexo g也会生成，不需要拷贝
11. db.json文件，不需要拷贝