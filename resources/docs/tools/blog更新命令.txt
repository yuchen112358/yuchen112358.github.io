####一、写好博客后，push到gihthub
```
git add -A    
git commit -m "begin blogs."         
git push origin master   
```

####二、转移到其他电脑从gihthub上pull到本地，保持本地与github网上同步
	git pull origin master
	
####三、特别注意：若提交的博客有语法错误，导致Jekyll不能编译通过，它便会发邮件给你，且本次提交的其他文档也不会编译，你的博客网页上不会产生任何更新。另外，即使你在本次更新时删除了_posts目录下的一些文档，但是由于你也提交了有问题的文档，因此这些被删除的文档依然会在你的博客网页上出现（因为没有更新）！！

####四、更改remote路径
	git remote rm origin
	git remote add origin git@github.com:yuchen112358/yuchen112358.github.com.git

在github上设置好ssh key后，使用ssh路径作为remote路径，即可使得每次push不需要再输入用户名和密码。同时注意，也可给自己的github账号设置多个ssh key，以便在不同的电脑上使用。
