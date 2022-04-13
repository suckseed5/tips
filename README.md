##################################################################################################
【1、docker部署】
##################################################################################################
* 【版本说明】
    - V1版本： 私有化版本，包括website+Django+tml环境+模型
* 【统一帐号】
    - account : tmlcloud 
    - pwd : TML!@#qaz
* 【V1】
    - Docker：
        (0) 创建容器： docker run -itd -p 9001:80 osint:latest /bin/bash   注意：一定要设置网络端口，否则无法通过网络访问
                      如果创建时忘记加，可如下解决https://www.cnblogs.com/chenpython123/p/10823879.html
        (1) 查看容器： docker ps -a
        (2) 启动容器:  docker start [CONTAINER ID] 即（1）中的id，例如docker start 7ecc6a84fb24
                      docker exec -it 7ecc6a84fb24 /bin/bash
        (3) 创建帐号:  useradd tmlcloud  && passwd tmlcloud  帐号密码为上述【统一帐号】
        (4) 创建新镜像：sudo docker commit -a "xly" -m "tml" 7ecc6a84fb24 customs:latest
                              （docker commit -a "[作者]" -m "[描述]" [docker_id] [新镜像名]:[新镜像版本]）
        (5) 导出：     docker save 01ce4b0b7695 > customs.tar
                              （docker save [image_id] > [customs.tar]）
        (6) 导入：     docker load -i customs.tar
        注意：1、安装docker  （注意不要安装在系统盘）
                    yum install docker
                    2、Docker 安装后 报 Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running? 解决办法
                    https://www.cnblogs.com/forturn/p/9371841.html
        (7) 进入容器：sudo docker run -it [image_name] /bin/bash
        (7.1)docker容器自启动：docker container update --restart=always $(docker ps -a -q)
        关闭自启动：docker container update --restart=no $(docker ps -a -q)
        (8) 进入容器启动服务
       命令：  sh /etc/rc.d/rc.local（把一些要单独启动的进程放在该目录下启动）
                     su - tmlcloud
                     （如果配置深度学习，即bio语料训练，需要到认知云帐号下起服务，tmlcloud里面才有深度学习环境，
                     1、bio语料训练也可以在该帐号下训练，命令：python /home/tmlcloud/run/tml-dl-1.0/tml_chunker/_chunker.pyc --mode train --train_data customs_extraction_bio.txt --test_data 01_15_bio_modified.txt.concepts --epoch 10 --eval_perl /home/tmlcloud/run/tml-dl-1.0/tml_chunker/conlleval_rev.pl
                     2、更新了tml模型后，可以直接将/home/tmlcloud/run/models/...目录下的bin文件替换掉，并再次执行sh和su命令）
                      ./bin/start.sh(注意用空格)（其他需要另外启动的服务）
                    nohup python /home/tmlcloud/NLP_文档分析/python/tml_service.tml -p 9090 &
      (9) 外部服务器测试
                    准备测试数据data.json，放在目录下：/data/NLP_test/python_test
                    命令：nginx
                                nohup python /data/NLP_test/python_test/test_client.py &
                    （查看nohup.out没有错误，即回传数据会保存在回传数据.json文件中）
      (10)可能遇到的一些问题：
           1、docker安装默认位置在系统盘，需要重新更改下位置，例：https://blog.csdn.net/runner668/article/details/80713713
           2、执行测试脚本后，空间快速沾满，一般docker需要空间10G左右
                 解决办法：查看tml是否授权成功（针对每台电脑有一个授权文件，docker也不能保存）
                 - tml环境配置：
                 准备_tml、_tml_server、_tml_client、tml_key.dat、_tml_getinfo（注意版本是centos版本，目录tml-kb/bin/centos-6.2-64版本）放到/usr/bin/目录下
                 在上述文件上传到放到docker里面，目录：/usr/bin/
                 将tml_key.dat、_tml_getinfo放到/etc目录下：
                 使用命令如下：
                 _tml_getinfo -key tml_key.dat -out tml_info.dat
                通过_tml_getinfo生成tml_info.dat，将tml_info.dat发给孟老师，返回给你tml_license.dat，再将tml_info.dat放到/etc目录下
                在/etc/路径下执行以下两个命令，赋权限：
                chmod +x tml_key.dat
                chmod +x tml_license.dat
            3、如果有深度学习语料，需要执行su - tmlcloud命令，在认知云帐号下启动服务，具体操作流程见上面
            4、/etc/rc.d/rc.local开机自启动可能用不了，只能手动sh /etc/rc.d/rc.local 
            5、可能需要替换/home/tmlcloud/develop/tml_platform-1.0/tml_backend/impls/backend_impl.pyc
            6、如果部署了认知云，需要把测试脚本settings.py,tml_cloud_settings.py,mapping.py与认知云参数相对应修改，特别注意tml_cloud_settings.py中ANALYZE_BY_APPKEY_URL需成本地地址，例：ANALYZE_BY_APPKEY_URL = "https://127.0.0.1/api/analyze_by_appkey/"
            7、重新部署一台服务器的时候，会出现服务器本身访问不了docker的情况
                原因1：需要修改test_client.py文件中的url改为该服务器的url
                原因2：服务器或者docker防火墙没有开
                原因3：docker配置了nginx后，外部服务器nginx没有配置docker的ip
                解决方法：vi /etc/nginx/nginx.conf
                                    将location这一块内容替换为，其中172.17.0.2与docker一致
                                        location /gmy_analyze {
                                            proxy_pass        http://172.17.0.2/gmy_analyze;
                                            proxy_redirect off;
                                            proxy_read_timeout 200;
                                            proxy_set_header  Host  $host;
                                            proxy_set_header  X-Real-IP  $remote_addr;
                                            proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
                                            proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
                                        }
                                        重启docker和nginx
    - Lib:
        (1) 系统库： yum install openssl openssh-server openssh* openssl-devel sudo gcc-c++ wget zlib-devel pcre pcre-devel \
                    boost-devel mysql* gperf libevent sqlite-devel postgresql-server.x86_64 sqlite* libevent libevent-devel libevent2 \
                    libuuid libuuid-devel
                    service sshd start
        (2) Nginx: 
        
                - 下载：wget http://nginx.org/download/nginx-1.17.5.tar.gz ;  
                
                - 编译安装： ./configure ; make && make install
                - 配置： /usr/local/nginx/conf/nginx.conf 
                        location / {
                            root /var/www/nginx-tml-platform/;
                            expires 5d;
                        }
                - 启动: /usr/local/nginx/sbin/nginx
                - 创建目录: mkdir -p /var/www/nginx-tml-platform && chown tmlcloud:tmlcloud -R /var/www
        (3) Mysql
                - 启动：   service mysqld start
                - 配置：   1. mysqladmin -u root password pku123
                          2. mysql -u root -p
                              create database rzy;    
                          3. vi /etc/my.cnf  
                             [mysqld]
                             datadir=/var/lib/mysql
                             socket=/var/lib/mysql/mysql.sock
                             user=mysql
                             symbolic-links=0
                             character-set-server = utf8
                             skip-grant-tables
                             [mysql]
                             default-character-set=utf8
                             [mysqld_safe]
                             log-error=/var/log/mysqld.log
                             pid-file=/var/run/mysqld/mysqld.pid       
        (6) gearmand
                - 下载: wget https://github.com/gearman/gearmand/releases/download/1.1.12.1/gearmand-1.1.12.1.tar.gz
                - 安装: gearmand-1.12
                - 编译安装： ./configure ; make && make install
                - 启动：gearmand -d
        (7) Twisted10.0(注意使用该版本，新版本有bug不建议使用)
                - 下载: wget https://twistedmatrix.com/Releases/Twisted/10.2/Twisted-10.2.0.tar.bz2
                - 安装: python setup.py build & python setup.py install
    - Web:
        (1) Django的settings.py 注意数据库的地址与账号密码需要检查，与上面mysql配置的一致;
            backend中的setting  上述两个setting默认使用settings.py.docker
        (2) python库的安装 
            a. pip freeze > requirements.txt 生成项目需要的依赖库文件
            b. pip install -r requirements.txt  进行安装
            c. pip install mysql MySQL-python simplejson
        (3) 同步数据库
            python manage.py makemigrations
            python manage.py migrate
        (4) gearman worker启动
            cd tml_platform : ./scripts/start_gearman_workers.sh
            cd tml_backend : ./scripts/start_gearman_workers.sh
        (5) 后端分析服务启动
            cd tml_backend: ./scripts/start_backend_service.sh 
    - rc.local
    - 测试：
        reboot检查ngixn与django是否已启动，同时*.*.*.*/cloud 看前端是否可以正常登录
##################################################################################################
【2、数据库相应命令（可以mysql界面化操作）】
##################################################################################################
1、登录查看：
	mysql -h datafactory.tmlsystem.com -P 52869 -u tao -ppku123
	show databases;
	use tml_crawler_db;
	show tables;
	SELECT * FROM spider_tasks_result WHERE main_sha1 LIKE '%f0aeeed06fa464fc54d91ba5ea9fbeafd441a74a%' and (spider_name LIKE 'articles_weibo' or spider_name LIKE 'articles_tieba');
2、增删改查：
	#增加一列vip，类型varchar默认为0，放在main_sha1后面：
	alter table spider_tasks_queue add column vip varchar(255) DEFAULT 0 after main_sha1;
	#删除一列vip
	ALTER TABLE spider_tasks_queue DROP COLUMN vip;
	#修改某个字段（直接执行有问题，直接界面操作）
    update chatbot.events set sender_id = 'purchase_luoyan.xie' where sender_id = 'jdpurchase_luoyan.xie6';
    #替换某个字段
    update chatbot.company_address set company_name = replace(company_name,"索尼中国","索尼（中国）") where company_name like "%索尼中国%";
    #随机查找数据
    select * from botstructure.bot111_train  where Intent = 'other' order by rand() limit 5;
    
3、卸载mysql https://blog.csdn.net/iehadoop/article/details/82961264
4、安装mysql https://www.cnblogs.com/opsprobe/p/9126864.html
      （1）、安装数据库：
        apt install mysql-server
      查看是否安装成功：
        netstat -tap | grep mysql
      进入数据库(默认没有密码)：
        mysql -u root -p
      修改密码：
        use mysql;
        UPDATE user SET password = PASSWORD('新密码') WHERE user = 'root'; 
        或者
        SET PASSWORD FOR 'root'@'%' = PASSWORD('新密码');
        或者
        update mysql.user set authentication_string=password('新密码') where user='root';
        FLUSH PRIVILEGES;
      数据库修改远程访问权限：
        use mysql;
        update user set host = '%' where user = 'root';
      查询mysql中所有用户权限：
        select host, user from user;
        FLUSH PRIVILEGES;
      mysql配置文件注释：
        vi /etc/mysql/mysql.conf.d/mysqld.cnf
      注释这一行：
        bind-address=127.0.0.1
      mysql重启：
        service mysqld restart
      （2）、安装数据库客户端
        apt-get install mysql-workbench
5、安装workbench sudo apt-get install mysql-workbench
6、数据库中文编码问题Er1366，解决方法：某个字段名，例如city，进入编辑页面，collation改为utf8_general_ci
7、id重新排序：
alter table chatbot.jd_express drop column id;
alter table chatbot.jd_express add id mediumint(8) not null primary key auto_increment first;
####################################################################################################
【3、SVN操作】
####################################################################################################
#SVN与git类似
sudo apt-get install subversion
svn co svn://10.0.0.3:9091/customers/dev_deployment/ --username tml--password mt12sadkjh!@
svn add
svn commint　-m
svn up
svnstatus 

###################################################################################################
【4、前后端调试步骤】
###################################################################################################
【h5】
  1、前期准备：
    1.1、将前端代码git到本地：git clone
    1.2、配置nodejs+npm+vue环境：https://www.cnblogs.com/HansBug/p/8901865.html
  2、切换到h5目录下面
  3、到配置目录切换测试环境：
  执行命令：
    cd cd config/
    ln -s index.js.product index.js
  注意：index.js不用提交
  4、本地安装依赖：
  执行命令：
    npm install
  5、启动项目：
  执行命令：
    npm run dev
  6、打开网址查看前端效果：http://localhost:8081
  7、提交代码

【web】
  1、进入测试环境：
  执行命令：
    cd scripts
    sh publish_dev_package.sh
  或者
    ssh web@demo.tmlsystem.com -p 8123
  2、进入测试服务器：需要配置登录权限，将公钥发给雪姐加下
  执行命令：
    cd develop/tml-osint-web
    git pull origin master
  3、打开测试网址：http://47.93.121.36/search/#/main/
  登录帐号、密码进行测试，例：shianbang  123456
  4、提交代码

【后端接口对接】
测试环境：ssh web@10.0.0.2
                   cd develop/yulongxue/tml-osint-factory/
例：前端雷达图对接
修改search_view.py中的category_of_radial_chart函数和urls.py中的url('category_of_radial_chart/', category_of_radial_chart),  # 网状图
调试：
注意：前端是 http://47.93.121.36/apps/category_of_radial_chart/   后端nginx配置的是http://10.0.0.2：9095/apps/category_of_radial_chart/
  1、在search_view.py中设置断点
  2、启动调试环境：python manage.py runserver 0.0.0.0:9095
  3、本地post请求：（post：新增，put：更新，get：查询，delete：删除）
        import requests
        data = {'time_delta': 10, 'report_sha1': 'e3d130b5d35154503db5cbfb5df184b3ab022700'}
        url = 'http://10.0.0.2:9095/apps/category_of_radial_chart/'
        r = requests.post(url,data)
        #r.text
  4、在服务器端可以断点测试
注意：get和post的区别在于：
1、get直接把参数包含在URL中，post通过request body传递参数；
2、get把http header和data一起同时发送，服务器响应200（返回数据），即产生一个TCP数据包；post浏览器先发送header，服务器响应100continue，浏览器在发送data，服务器响应200（返回数据），即产生两个TCP数据包。
##################################################################################################
【5、前端的一些知识】
##################################################################################################
1、html网页大概框架；vue网页详细呈现内容（长什么样）；js在vue中的点击事件所用到的详细方法；css样式的定义
2、vue框架eliment-ui,js弹出层组件layer
3、弹出层组件layer(样式适应性极高)替代传统的alert(样式旧，字体无法加颜色)
4、jQery方法：
方法一：点击事件后若请求失败先弹出提示信息再弹出对话框
$(document).ready(function() {
  $('#myCustomTrigger').click(function (event) {
jQuery.ajax({
  url: "<url>",   
  type: "get",
  cache: true,
  dataType: "script",
        error: function(XMLHttpRequest, textStatus, errorThrown) {
              //需要下载该组件
              layer.open({
                title: '提示',
                content: '<span>首次登陆烦请您先进行反馈表安全验证。具体方法为点击“确认”按钮，访问“https://43.82.117.204/”，</span><span style="color:#ff0000;">点击“高级”，点击“继续前往”</span><span>。之后请您</span><span style="color:#ff0000;">刷新本页面</span><span>，点击工单申请按钮，提供您的申请。谢谢！</span>',
                btn: ['确认', '取消'],
                        yes: function(index, layero) {
                            window.open('https://43.82.117.204/')
                            parent.layer.close(index);//关闭弹出层
                            // window.parent.location.reload();//刷新父页面
                        },
              });
          }
});
window.ATL_JQ_PAGE_PROPS =  {
  "triggerFunction": function(showCollectorDialog) {  
      showCollectorDialog();
  }};
 });
});
XXXXXX方法二：页面加载时若请求失败先弹出提示信息，点击事件后再弹出对话框
jQuery.ajax({
  url: "<url>",  
  type: "get",
  cache: true,
  dataType: "script",
        error: function(XMLHttpRequest, textStatus, errorThrown) {
              //需要下载该组件
              layer.open({
                title: '提示',
                content: '<span>首次登陆烦请您先进行反馈表安全验证。具体方法为点击“确认”按钮，访问“https://43.82.117.204/”，</span><span style="color:#ff0000;">点击“高级”，点击“继续前往”</span><span>。之后请您</span><span style="color:#ff0000;">刷新本页面</span><span>，点击工单申请按钮，提供您的申请。谢谢！</span>',
                btn: ['确认', '取消'],
                        yes: function(index, layero) {
                            window.open('https://43.82.117.204/')
                            parent.layer.close(index);//关闭弹出层
                            // window.parent.location.reload();//刷新父页面
                        },
              });
          }
});

window.ATL_JQ_PAGE_PROPS =  {
  "triggerFunction": function(showCollectorDialog) {  
    jQuery("#myCustomTrigger").click(function(e) {
    e.preventDefault();
      showCollectorDialog();
    });
  }}
5、mounted()：刷新页面执行的操作
6、localStorage：长久保存整个网站的数据,保存的数据没有过期时间,直到手动去删除
插入字段：localStorage.setItem('username',processName);
获取字段：localStorage.getItem("username");
删除字段：localStorage.removeItem('username');
7、VueI18n：中英文切换
8、el-table
（1）表头固定：设置max-height
（2）调整表头高度
  .el-table__header tr,
  .el-table__header th {
    padding: 0;
    height: 42px;
    font-size: 14px;
  }
（3）调整表格高度
  .el-table__body tr,
  .el-table__body td {
    padding: 0;
    height: 80px;
    font-size: 14px;
  }
8.1、el-table-column
单元格字数过长时省略：show-overflow-tooltip
9、vue中处理数据，可以在method中写处理函数，return处理后的值，vue直接调该函数
<div v-for="item in splitFunction(scope.row.image)">
<a :href="imageURL+item" target="_blank" style="color: #409eff">{{item}}</a>
splitFunction(data){
  if (data != undefined){
    var imageList = data.split(",")
  }
  return imageList
}
10、静态文件超链接(a标签)，实现点击静态文件，浏览器可以进行查看，该静态文件可以放在static目录下，也可以放在对应服务器其他目录，但需要配置nginx，例如：url=http://192.168.11.103/botimg/trip05.png
11、前端上传文件给后端：el-upload
12、前端下载文件：后端返回文件的url（nginx配置），前端打开location.href=(url);
13、布局el-row行(设置每列之间的距离)：<el-row :gutter="24">，el-col列(设置几行)：<el-col :span="10">
14、可以直接在js中构建html代码字符串，实现展示；或者，先vue实现展示，js中加判断条件，当条件满足时，vue实现展示
15、若需要将文件(上传时需为二进制)和json值都作为参数传给接口，可以定义new FormData()
16、若报warning，该函数没有这个方法，可以打印该函数检查，可能该函数是列表，需要[i]才能取值
17、element-ui中英文切换

#main.js
import Element from 'element-ui'
import ElementLocale from 'element-ui/lib/locale'
import 'element-ui/lib/theme-chalk/index.css'
import i18n from './lang'
Vue.use(Element)
// element-ui组件中英文切换的关键
ElementLocale.i18n((key, value) => i18n.t(key, value))
/* eslint-disable no-new */
new Vue({
  el: '#app',
  router,
  i18n,
  components: { App },
  template: '<App/>'
})
#lang/index.js
import zh from './zh.js';
import en from './en.js';
import Vue from 'vue'
import VueI18n from 'vue-i18n'
import enLocale from 'element-ui/lib/locale/lang/en'
import zhLocale from 'element-ui/lib/locale/lang/zh-CN'
Vue.use(VueI18n);
const messages = {
  'zh':{
    ...zh,
    ...zhLocale
  },
  'en':{
    ...en,
    ...enLocale
  }
}
const i18n = new VueI18n({
  // 不设置默认语言，语言状态由历史记录决定(主页面每次切换语言的时候，需要将语言状态存到localStorage中)
  locale: localStorage.getItem('lang')||'zh',
  messages
})
export default i18n;
#lang/en.js
import {XX} from "../utils/url.js";
export default {
  sendNullMessage: "Can't send empty content"
}
#lang/en.js
import {XX} from "../utils/url.js"
export default {
  sendNullMessage: "不能发送空的内容"
}
#实际使用
#定义
sendNullMessage: this.$t("sendNullMessage")
#vue方法中
$t("sendNullMessage")
#js方法中
this.$t("sendNullMessage")
18、修改组件默认style——!important
.el-form-item__label {
  font-size: 12px !important;
}
19、前端调样式的时候，若出现怎么调也没用的情况，可能是项目默认css文件中配置了
20、单独html实现button，调用api功能，可以使用$.ajax
21、前端项目文件：router-路由配置文件夹（ip/detail;ip/detail1）跳转
                路由直接带参数：
                      {
                      path: '/detail/:id/:info/:isTrue/:classList',
                      name: 'detail',
                      component: Detail
                      }
                调用:this.$route.params
###################################################################################################
【6、linux的一些命令】
###################################################################################################
（1）#!/bin/sh
ps -ef|grep python|grep -v grep|awk '{print "kill -9 "$2}'|sh
（2）sudo加上密码
echo %s|sudo -S %s' % (password,command)
（3）杀掉占用指定端口服务:sudo kill -9 $(lsof -t -i:5003)
（4）查询历史记录:history|grep super 
（5）查看内存:top 
（6）查看文件位置:locate _tml_getinfo 
（7）文件名:find -iname 
（8）查看有多少文件：ls -l
（9）列出文件夹中*2019_11_14*文件的个数：ls -l *2019_11_14*|grep "_"|wc -l
（10）删除a.txt文件中的1-100行：sed -i '1,100d' a.txt
（11）查找出现指定内容的文件：grep -nIr "127.0.0.1"
（12）查看端口占用：sudo netstat -anp |grep 80
###################################################################################################
【7、爬虫知识】
###################################################################################################
1.摘要值：（类似于文件的标签，来确定文件在传输过程中是否是完整的）
保证数据的完整性：例如你发送一个100M的文件给你的B，但是你不知道B收到的是否是完整的文件；此时你首先使用摘要算法，如MD5，计算了一个固定长度的摘要，将这个摘要和文件一起发送给B，B接收完文件之后，同样使用MD5计算摘要，如果B计算的结果和你发送给他的摘要结果一致，说明B接收的文件是完整的。
2.MD5(加密)：MD5算法就像一个函数，任意一个二进制串都可以作为自变量进入这个“函数”，然后会出来一个固定为128位的二进制串。
3.Hash函数是一个将任意长度的消息(message)映射成固定长度消息的函数 简称h（X）对于任何消息x ,将h(x)称为x的Hash值、散列值、消息摘要值。
注：MD5是Hash函数的一种
###################################################################################################
【8、pip镜像/源下载】
###################################################################################################
1、sudo pip install solrpy -i http://pypi.douban.com/simple --trusted-host pypi.douban.com
yum（适用于centos）
2、pip安装方法
全自动安装： easy_install jieba 或者 pip install jieba / pip3 install jieba
半自动安装：先下载 https://pypi.python.org/pypi/jieba/ ，解压后运行 python setup.py install
手动安装：将 jieba 目录放置于当前目录或者 site-packages 目录
通过 import jieba 来引用
#################################################################################################
【9、虚拟环境安装】
#################################################################################################
1、virtualenv
https://www.cnblogs.com/freely/p/8022923.html（python2.6安装存在问题，可以重装python2.7）
2、conda:额外安装其他包
#################################################################################################
【10、python知识】
#################################################################################################
1、编码问题
      /xe转成中文
      例：li = [((33, 39), '宝马'), ((36, 39), '马')]
              print str(li).decode("string_escape")
      输出就是可查看的样式 [((33, 39), '宝马'), ((36, 39), '马')]
2、全角问题
      import re,pdb
      #解决字符串不能编译成unicode的问题
      import sys
      reload(sys)
      sys.setdefaultencoding('utf8')
      s = '无限极（中国）有限公司'
      ss = re.sub(u'[（]', '(', s.decode())
      aa = re.sub(u'[）]', ')', ss.decode())
      print aa
3、break与continue问题（必须配合if语句使用）
      break:循环中，若判断满足条件，执行break后所有循环都停止(结束所有循环)
      continue:循环中，若判断满足条件，执行continue后，以下代码不执行，继续下次循环(提前结束本轮循环，并直接开始下一轮循环)
4、类的例子
      （1）简单的例子
      class Student(object):
        def __init__(self,name,score):
          self.name = name
          self.score = score
        def get_score(self):
          if self.score < 60:
            return '该生不及格'
          elif 60 <= self.score <= 100:
            return '该生及格了'

      lisa = Student('Lisa',90)
      print(lisa.name,lisa.get_score())
      （2）访问限制
      上面的例子中，虽然内部定义了name和score，但是外部还是可以修改，例：
      >>> bart = Student('Bart Simpson', 59)
      >>> bart.score
      59
      >>> bart.score = 99
      >>> bart.score
      99
      若改成self.__name = name、self.__score = score，就变成了私有变量，只有内部可以访问，外部不能访问
      （3）继承和多态
      I、继承
      上面例子中object表示所有类，当然我们可以定义基类/父类，让子类继承，例：
      ##父类
      class Animal(object):
          def run(self):
              print('Animal is running...')
      ##继承的子类
      class Dog(Animal):
          pass  
      >>>>Animal is running...
      II、多态
      当父类和子类存在相同的方法时，父类的方法会被覆盖
      ##继承的子类
      class Dog(Animal):
          def run(self):
              print('Dog is running...') 
      >>>>Dog is running...  
      III、实例属性和类属性
      ##实例属性
      class Student(object):
        def __init__(self,name,score):
          self.name = name
          self.score = score
      ##类属性(归student所有)
      class Student(object):
        name = 'student'
      （4）使用__slots__限制实例的属性
      class Student(object):
          __slots__ = ('name', 'age') # 用tuple定义允许绑定的属性名称
      （5）@property，对类中的参数进行必要的检查，例：
      class Student(object):
          #将score方法转换成score属性，可属性直接调用a=Student.score而不是b=Student.score()
          @property
          def score(self):
              return self._score
          #直接给属性赋值，Student.score='hh'而不是Student.score=('hh')
          @score.setter
          def score(self, value):
              if not isinstance(value, int):
                  raise ValueError('score must be an integer!')
              if value < 0 or value > 100:
                  raise ValueError('score must between 0 ~ 100!')
              self._score = value
      >>> s = Student()
      >>> s.score = 60 # OK，实际转化为s.set_score(60)
      >>> s.score # OK，实际转化为s.get_score()
      60
      >>> s.score = 9999
      Traceback (most recent call last):
        ...
      ValueError: score must between 0 ~ 100!
      即表示只允许对Student实例添加name和age属性
5、生成器
      列表生成式：a = [i for i in [1,2,3] if i > 3]
      占用空间大，取数据：a[1]....
      生成器：a = (i for i in [1,2,3] if i > 3)
      列表元素是按某种算法推算出来的，占用空间小，取数据：for
6、map函数（节省计算步骤）
      例：
      def f(x):
        return x*x
      r = map(f,[1,3,2,5])
      list[r]
      >>>>>[1,9,4,25]
7、匿名函数（lambda）：即不用另外定义一个函数来计算
      lambda n:n%2==1 等同于def test(n): return n%2==1
      L = list(filter(lambda n:n%2==1, range(1, 20)))(filter过滤range中满足lambda的值)
      >>>>>[1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
8、装饰器：（@）在不修改原有函数的技术，在代码运行期间动态增加功能的方式
      def log(func):
          def wrapper(*args, **kw):
              print('call %s():' % func.__name__)
              return func(*args, **kw)
          return wrapper
      @log
      def now():
          print('2015-3-25')
      now()
      >>>>>call now():
      >>>>>2015-3-25
      其中*args表示非关键字参数（以元祖形式呈现），**kw表示关键字参数（以字典形式呈现）
9、偏函数
      （1）转换为数字：int('123') ----->>>123
      （2）转换成二进制(base)：int('123',base=8)----->>>83
      （3）int2 = functools.partial(int, base=2)  int2('1000000')----->>>64
10、作用域
        类似__xxx__这样的变量是特殊变量，可以被直接引用，但是有特殊用途
        类似_xxx和__xxx这样的函数或变量就是非公开的（private），不应该被直接引用
11、排序
        (1)、list.sort(reverse=True)
        对原来的列表进行的操作，不会产生一个新列表
        (2)、sorted(list,reverse=True)
        在原来列表的基础上，产生一个有序的新列表，可以复制一个列表名
12、yield(相当于return,只有遇到next才会执行)
        import pdb
        def foo():
            print("starting...")
            while True:
                res = yield 4
                print("res:",res)
        # 直接调用含yield的函数，即foo不会真正执行，只是先得到一个生成器g(相当于一个对象)
        g = foo()
        >>>生成器
        # 当调用next函数时，返回yield的值，类似于return，即return后面的内容不执行(即也没有进行赋值)
        print(next(g))
        >>>starting...
        >>>4
        print("*"*20)
        # 当再次调用next函数时，会从上次停止的地方执行，因为上次没有赋值，所以res为None，执行循环，执行yield，输出4
        print(next(g))
        >>>res:None
        >>>4
13、静态方法：@staticmethod或@classmethod
      （1）相同：可以不需要实例化，直接类名.方法名()来调用；
      （2）不同：
            class A(object):  
                bar = 1  
                def foo(self):  
                    print 'foo'  

                @staticmethod  
                def static_foo():  
                    print 'static_foo'  
                    print A.bar  

                @classmethod  
                def class_foo(cls):  
                    print 'class_foo'  
                    print cls.bar  
                    cls().foo()  
            ###执行  
            A.static_foo()  
            A.class_foo()  
            >>>输出：
                static_foo
                1
                class_foo
                1
                foo
14、jira用法
        1、登录：（最好配置api token作为登录密码）
        # 通过jira域名和账户密码登录（'verify': False 解决jira报certificate verify failed的问题）
        jira = JIRA(SETTINGS.BASE_URL, basic_auth=(SETTINGS.USERNAME, SETTINGS.PASSWORD),options={'verify': False})
        2、获取每个项目的所有jira
        all_bug=jira.search_issues("project = 'WSD Worksheet System'",maxResults=-1)#jql语法，查询项目为JD下面的所有缺陷，最多2000条
        3、获取单个jira的issue
        issue = jira.issue(jira_id)
        4、新建jira
        create_jira = jira.create_issue(issue_dict)
        5、updatejira(增改删[{"add/set/remove":{"name":components_value}}])
        issue.update(components=[{"add":{"name":components_value}}])
        6、上传附件
        jira.add_attachment(issue=issue, attachment=attachmentFileName)
15、python收发邮件（smtplib/poplib）
        1、确保该邮箱的smtp/pop邮件服务器及port是多少，支持收发邮件
            telnet smtp.office365.com 587/telnet outlook.office365.com 110
            可以输出正确的登录提示
        2、配置邮箱的smtp/pop授权码
        3、发送html邮件，html格式/button应邮件服务器而显示不同，建议用打开外部链接的方式实现button功能
###############################################################################################
【11、正则】
#################################################################################################
(1)
.*：贪婪匹配（从最后面往前匹配）若字符串中包含多个需要匹配的内容，匹配一个完整的内容
.*?：懒惰匹配（从前面往后匹配）若字符串中包含多个需要匹配的内容，把多个内容都匹配出来
    例：101000000000100
    .*：1010000000001
    .*?：101
(2)找到不含title的句子：
    ^((?!title).)*$\n
(3)句子开头^，句子结尾$
    ^[0-9]|[1-9]{1,2}|[1-9][0-9]{1,3}|[1-9]+|\d+|100岁$
    注意：
    |是或的意思;
     ^表示正则开始;
     $表示正则结束;
     [0-9]表示0-9的数字等同于\d;
     [1-9]{1,2}表示十位和个位都是1-9的两位数;
     [1-9][0-9]{1,3}表示最最左边一位由1-9组成，右边是由0-9组成的1-3位数;
     [1-9]+其中的+号就表示{1,}有大于等于1位数。
(4)或者匹配
    (?:hello|hi)
(5)模式匹配符
    'adjhaajj' 将(.*)aa(.*)替换为$2aa$1  结果为'jjaaadjh'
(6)引用符
    '334778886' 用正则([0-9])(\1*) 找出['33','4','77','888','6']   （\1表示引用第一个分组）
(7)正则拆分:加上'()'，保留正则匹配的数据
    '长征宣告了[帝国主义](CON)和[蒋介石](PER)围追堵截的破产'，正则拆分：
    re.split(r'\[.*?\]\([A-Za-z]{1,5}\)',a)-----》  ['长征宣告了', '和', '围追堵截的破产']
    re.split(r'(\[.*?\]\([A-Za-z]{1,5}\))',a)-----》['长征宣告了', '[帝国主义](CON)', '和', '[蒋介石](PER)', '围追堵截的破产']
(8)()括号表示正则匹配出来的内容
'[帝国主义](CON)'需要匹配出'CON'，我们可以列出满足的规则，因为只需要匹配出'CON'，所以在需要输出的字符外面加上'()'
re.findall(r'\[.*?\]\(([A-Za-z]{1,5})\)')------》'CON'
(9)?:、?=、?<
exp1(?=exp2)：查找 exp2 前面的 exp1。
(?<=exp2)exp1：查找 exp2 后面的 exp1。
exp1(?!exp2)：查找后面不是 exp2 的 exp1。
#################################################################################################
【12、ubuntu一些安装注意点】
#################################################################################################
1、安装百度网盘(浏览器装插件baiduexporter，安装aria2)
https://github.com/acgotaku/BaiduExporter/tree/edge
pianshen.com/article/8104276906/
2、U盘删除文件显示：rm: cannot remove ' Read-only file system
查看u盘分区：df -h
进入root：sudo su
修复：fsck -y /dev/sdb4
关掉上面终端，新起终端
umount /media/niko/F2F6-1BE4
重插U盘
mount /dev/sdc4 /media/niko/F2F6-1BE4
2.1、U盘插入显示Error mounting
列出所有的分区情况：sudo fdisk -l
修复挂载错误的相应的分区：sudo ntfsfix /dev/sdb1
3、安装wine实现ubuntu上安装windows软件
安装wine: sudo apt-get install wine
卸载wine:sudo apt-get remove wine
        sudo rm -r /home/niko/.wine
        sudo apt-get autoremove
wine使用：下载某个软件的exe文件，wine ...exe
4、重装系统
    （1）linux制作ISO镜像U盘启动盘
         https://blog.csdn.net/master5512/article/details/69055662
    （2）插入U盘，按F12进入安装流程，选择usb模式
    （3）重装系统配置开发环境
        a、chrome
        https://blog.csdn.net/sinolover/article/details/94380049
        b、sublime
        https://blog.csdn.net/hzlarm/article/details/99675627
        https://www.yebaike.com/14/202104/3012781.html
        c、vscode
        settings.json文件(您可以在文件>首选项>设置>功能>终端>集成>自动化外壳:Linux中访问的文件)有一个参数 "terminal.integrated.inheritEnv": false 
        d、Aconda
        https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/
        e、java1.8
        https://blog.csdn.net/weixx3/article/details/94109337
        f、maven
        https://blog.csdn.net/FungLi_notLove/article/details/104469940
        https://blog.csdn.net/sublime_k/article/details/106801068
        https://www.cnblogs.com/xiaostudy/p/11664076.html
###########################################################################################
【13、tersorflow和pytorch的区别】
#################################################################################################
tersorflow：静态框架，构建完计算图后，既无法随时修改计算流程，不够灵活
pytorch：动态框架，对变量做任何操作都是灵活的
【（1）深度学习相关知识】
含义：是机器学习的子集，没有数学方法的支撑，属于神经网络构造不同的层次，直接跳过人为提取特征的过程，自动获取特征，构建模型，进行预测；适应于数据量大的实例。
学习率：（损失函数最小时，损失函数中的斜率函数的斜率）
  针对原始的样本（1,2）
  针对线性回归的函数 y=kx
  对应的损失函数是 y=2kx^2,
  那我们的方向就是希望当损失函数最小时，得到最终的k值，然后再代入到原始的线性函数中，那具体应该如何在最小化损失函数的时候得到对应的k值呢？
  方法一：对于损失函数求导，然后令导数等于0，得到对应的k值，有时候并不能直接解出来，并且这种方式可能是局部最优；
  方法二：采用梯度下降（斜率不断下降）与学习率的方法去求得最后的k值，明确梯度下降中的梯度
  实际指的是损失函数的斜率，初始对于k设定一个值例如0.3，然后将k值与样本中的x值代入到损失函数中，得到损失函数的y值就是差距值，如果这个差距值符合要求就可以，但是太大的话可能就需要不断的去调节这个k值，那新的k值如何获得呢，对应的公式如下：
  k1=k+at，
  其中k1就是新的k值，k是初始设定的那个k值，而其中的a就是学习率，一般可以设定0.01，对于学习率的设定，如果设定的太小就会导致最终收敛太慢，而如果设定的太大的话，可能就会错过最小值点，因此需要设定合适，而其中的t就是对应算是函数的斜率，得到的方式就是对损失函数进行求导，然后将样本中的x值与初始k值代入到对应的其中得到斜率，得到新的k值，然后再将新的k值和x值代入到损失函数中，看下函数的差值是否在那个区间内。
  总结：梯度下降其实就是斜率不断的下降，最终希望是斜率为0对应的就是在谷底的时候得到对应的k值，就是最好的k值。
【（2）机器学习相关知识】
含义：通过人为提取的特征构建模型达到预测的功能，例如：给出大量人员的身高和体重，构建相应的模型，输入一个身高来预测体重；
适应于数据量少的实例。
【（3）tensorflow一些知识】
1、tf.placeholder(dtype,shape=None,name=None)
含义：占位符，提前定义数据，分配内存；等建立session，在会话中，运行模型的时候通过feed_dict()函数向占位符喂入数据。
参数：
dtype：数据类型。常用的是tf.float32,tf.float64等数值类型
shape：数据形状。默认是None，就是一维值，也可以是多维（比如[2,3], [None, 3]表示列是3，行不定）
name：名称
2、tf.Variable()
含义：用于生成一个初始值为initial-value的变量；必须指定初始化值。
3、tf.get_variable()
含义：获取已存在的变量(要求不仅名字，而且初始化方法等各个参数都一样)，如果不存在，就新建一个；可以用各种初始化方法，不用明确指定值。
4、tf.nn.embedding_lookup(embedding, self.input_x)
含义：查表：根据input_x中的id查找embedding中对应的元素
5、tf.name_scope(name=None)
含义：指定的name区域中定义的所有对象及各种操作，他们的“name”属性上会增加该命名区的区域名，用以区别对象属于哪个区域
6、tf.layers.conv1d()
含义：一维卷积层
7、tf.reduce_max()
含义：求最大值
8、tf.layers.dense()
含义：全连接层
9、tf.contrib.layers.dropout()
含义：dropout防止过拟合，随机的拿掉网络中的部分神经元，从而减小对W权重的依赖，以达到减小过拟合的效果。
10、tf.nn.relu()
含义：relu激活-线性整流函数，大于0的值呈线性，小于0的值保持为0不变
11、tf.argmax()
含义：按行或列返回最大值的索引
12、tf.softmax()
含义：以某一个轴的下标为索引，对这一轴上其他维度的值进行 激活 + 归一化处理
13、tf.nn.softmax_cross_entropy_with_logits()
含义：
14、tf.reduce_mean()
含义：用于计算张量tensor沿着指定的数轴（tensor的某一维度）上的的平均值，主要用作降维或者计算tensor（图像）的平均值。
15、tf.train.AdamOptimizer()
含义：实现Adam算法的优化器
16、tf.equal()
含义：对比这两个矩阵或者向量的相等的元素，如果是相等的那就返回True，反正返回False，返回的值的矩阵维度和A是一样的
17、tf.summary.scalar
含义：用来显示标量信息，一般在画loss,accuary时会用到这个函数。
18、tf.summary.merge_all()
含义：可以将所有summary全部保存到磁盘，以便tensorboard显示
19、tf.summary.FileWriter
含义：将所有summary写入指定目录
20、tensorflow加载预训练模型和保存模型（ckpt文件）
|--checkpoint_dir
|    |--checkpoint
|    |--MyModel.meta
|    |--MyModel.data-00000-of-00001
|    |--MyModel.index
（1）、MyModel.meta文件保存的是图结构，meta文件是pb（protocol buffer）格式文件，包含变量、op、集合等。
（2）、ckpt文件是二进制文件，保存了所有的weights、biases、gradients等变量。存在MyModel.data-00000-of-00001和MyModel.index中。
（3）、保存tensorflow模型
      saver = tf.train.Saver()
      saver.save(sess,"./checkpoint_dir/MyModel")
（4）、导入训练的模型
      import tensorflow as tf
      with tf.Session() as sess:
        new_saver = tf.train.import_meta_graph('./checkpoint_dir/MyModel-1000.meta')
        new_saver.restore(sess, tf.train.latest_checkpoint('./checkpoint_dir'))
#################################################################################################
【14、neo4j搭建教程-构建知识图谱】
#################################################################################################
步骤一：neo4j ubuntu安装环境配置，见：blog.csdn.net/qq_32507417/article/details/112403721
    1、安装java1.8
    2、安装neo4j：curl -O http://dist.neo4j.org/neo4j-community-3.4.5-unix.tar.gz
                 tar -axvf neo4j-community-3.4.5-unix.tar.gz
                  cd neo4j-community-3.4.5-unix
                  vim conf/neo4j.conf
                    修改：# 修改第22行load csv时l路径，在前面加个#，可从任意路径读取文件
                          #dbms.directories.import=import

                          # 修改54行，去掉改行的#，可以远程通过ip访问neo4j数据库
                          dbms.connectors.default_listen_address=0.0.0.0

                          # 默认 bolt端口是7687，http端口是7474，https关口是7473，不修改下面3项也可以
                          # 修改71行，去掉#，设置http端口为7687，端口可以自定义，只要不和其他端口冲突就行
                          dbms.connector.bolt.listen_address=:7687

                          # 修改75行，去掉#，设置http端口为7474，端口可以自定义，只要不和其他端口冲突就行
                          dbms.connector.http.listen_address=:7474

                          # 修改79行，去掉#，设置http端口为7473，端口可以自定义，只要不和其他端口冲突就行
                          dbms.connector.https.listen_address=:7473

                          # 修改验证，去掉#
                          dbms.security.auth_enabled=false

                  启动、控制台、停止服务:bin/neo4j start // console  // stop
步骤二：neo4j和neo4j-driver安装-见实例：neo4j_test.py
       pip install neo4j==4.2.1 neo4j-driver==1.7.6

步骤三：py2neo-见实例：py2neo_test.py
        pip install py2neo==4.3.0
        
注：步骤二和三可选一，都是以一为基准
知识点：
1、关系数据库(mysql数据库)与图数据库(neo4j),图数据库按照节点显示，性能更快
2、neo4j的一些命令：（neo4j/py2neo）https://blog.csdn.net/leisure_life/article/details/121120511
(1)创建带标签和属性的节点：CREATE (n:Person { name: 'Andres', title: 'Developer' })
   创建节点间的关系：MERGE (song)-[:SUNG_BY]->(singer)
(2)删除：MATCH (n:Person { name: 'UNKNOWN' }) DELETE n
(3)修改：MATCH (n:Person { name: 'UNKNOWN' }) SET n.name = '张三'
(3)查(节点'()',关系'[]')：
      MATCH：MATCH (a:Song{name:"后来"})--(b:Singer) RETURN b.name查找name属性为“后来”的节点Song,与之关系的Singer节点值b，返回b的name
      MERGE：MERGE (song:Song {id:"1", name:"后来"})
      MERGE = CREATE + MATCH，如果它不存在于图中，则它创建新的节点/关系并返回结果
#################################################################################################
【15、rasa知识点】
#################################################################################################
（1）rasa的开源UI-聊天工具 （见project/chatbot-UI/chatroom、test_bot）】
    1、在已有的rasa bot项目中新建credentials.yml，在其中加上内容：
    rest:
    2、启动bot:rasa run --cors "*" --enable-api -vv --log-file trip.log -p 5005 &
    3、搭建chatroom环境：github.com/scalableminds/chatroom
       (1).修改index.html中的端口号与2中启动的端口号一致
       (2).yarn install
       (3).yarn watch(启动后就可以进行下一步了，只要没有error就行)
       (4).yarn serve
    4、查看聊天室：http://localhost:8080
（2）rasa模型建立的一些经验----return [FollowupAction("action_listen")]
    1、config：
    N-Grams：按长度 N 切分原词得到的词段，例：https://www.zhihu.com/question/357850262/answer/910808076
    JiebaTokenizer：jieba分词
    DIETClassifier：若需要用实体的同义词，需要设置entity_recognition为false
    - name: JiebaTokenizer
          ---作用：中文jieba分词
          ---输入：字符串
          ---输出：tokens--jieba分成的词语
    - name: RegexFeaturizer
          ---作用：通过正则表达式进行意图分类
          ---输入：tokens
          ---输出：用户信息和token的稀疏特征
    - name: CRFEntityExtractor
          ---作用：条件随机场（CRF）实体提取
          ---输入：tokens和dense_features (密集功能)
          ---输出：entities
    - name: EntitySynonymMapper
          ---作用：实体同义词
          ---输入：现有实体
          ---输出：已知实体同义词
    - name: CountVectorsFeaturizer
      analyzer: char_wb
      min_ngram: 1
      max_ngram: 4
          ---作用：用户信息和token的稀疏特征
          ---输入：tokens
          ---输出：tokens
    - name: DIETClassifier
      entity_recognition: False
      epochs: 100
          ---作用：用于意图分类和实体提取的双意图实体转换器(DIET)
          ---输入：用户消息以及可选的意图的density_features(密集特征)和/或sparse_features(稀疏特征)
          ---输出：entities, intent and intent_ranking(意图排名)
    - name: ResponseSelector
      epochs: 100
          ---作用：响应选择器
          ---输入：用户消息以及可选的意图的density_features(密集特征)和/或sparse_features(稀疏特征)
          ---输出：用键作为响应选择器的检索意图的字典，值包含检索意图下预测的响应模板、置信度和响应键
    - name: FallbackClassifier
      threshold: 0.25
      ambiguity_threshold: 0.1
          ---作用：如果 NLU 意图分类评分不明确，则使用 NLU _ fallback 意图对消息进行分类。
          ---输入：意图和意图从以前的意图分类器中排序输出
          ---输出：entities, intent and intent_ranking(意图排名)
    policies:
    - name: MemoizationPolicy
      max_history: 4
          ---作用：记忆政策会记住你训练数据中的story。它检查当前对话是否与训练数据中的story相匹配。如果是这样的话，它将以1.0的信心从你的训练数据的匹配story中预测下一步行动。如果没有找到匹配的对话，该政策预测没有，信心为0.0。在训练数据中查找匹配项时，该策略将考虑对话的最后max_history轮数。 一轮包括用户发送的消息以及助手在等待下一条消息之前执行的任何操作。max_history表示查看多少历史对话记录
    - name: TEDPolicy
      max_history: 4
      epochs: 100
          ---作用：变压器嵌入式对话(TED)政策
    - name: RulePolicy
      core_fallback_threshold: 0.25
      core_fallback_action_name: action_default_fallback
          ---作用：RulePolicy是一种处理遵循固定行为（例如业务逻辑）的对话部分的策略。 它根据您的训练数据中的rules进行预测。
    2、一些方法rasa.shared.core.event
    AllSlotsReset---reset_slots
    所有插槽均重置为其初始值。如果您想保留对话历史记录而只想重置插槽

    SlotSet---slot
    插入词槽，例：SlotSet(entity_way_,None)

    Restarted---restart
    对话应该重新开始并清除历史记录。除了删除所有事件，此事件可用于重置跟踪器状态（例如，忽略任何过去的用户消息并重置所有插槽）。

    session_started
    标记新会话的开始。

    resume
    机器人接管了对话。

    undo
    Bot撤消其最后的动作。
    举例：若把restart加在rule里面--->匹配完正确的答案后，还会返回action_default_fallback的答案
    3、rules的一些测试
    utter
    sdk-restart  出现正确答案(有很多None)
    restart    出现正确答案和default(无None)  
    sdk-restart+restart  出现正确答案偶尔出现default(无None)
    restart+sdk-restart  出现正确答案和default(无None)
    action
    sdk-restart 出现正确答案(有很多None)
    restart   出现正确答案和default(无None)
    sdk-restart+restart 出现正确答案和default（无None）
    restart+sdk-restart  出现正确答案和default（无None）
    #######################################
    无default
    utter
    restart    出现正确答案(有None)
    action
    restart   出现正确答案(有None)
    ####################################################
    4、解决action_listen没有答案返回的问题
    （1）只用rulepolicy,不用TEDpolicy,每个rules后面加上restart---没有空答案，但是会出现正确答案和default答案
    （2）只用rulepolicy,不用TEDpolicy,每个rules后面加上wait_for_user_input: False--没有空答案，但是会出现正确答案和default答案
    5、rasa源码的一些修改
    （1）为解决jieba分词实体标注错误的问题，详见project/rasa_code_change/jieba_tokenizer.py
    修改rasa/nlu/tokenizers/jieba_tokenizer.py
    （2）问句标点符号的问题(若标点符号不影响意图区分)
        a.训练数据nlu可以去除末尾所有标点符号再训练，等同于修改rasa/shared/nlu/training_data/message.py line line 116 （该文件是训练数据输入处，在所有config处理之前）
        text = text.strip('?').strip('!').strip('？').strip('！')

        b.测试数据/问句可以去除末尾所有标点符号再重启服务（标点符号的不同就不会导致答案不一样）
        b.1.修改rasa/core/processor.py line467(这不会导致events中是去除标点的数据)
        # 对输入问句处理掉末尾标点符号，但为了样本导出方便统计，存入events还是原来的数据
        rm_punctuation = message.text.strip('?').strip('!').strip('？').strip('！')
        if self.message_preprocessor is not None:
            # 原来的代码
            #text = self.message_preprocessor(message.text)
            text = self.message_preprocessor(rm_punctuation)
        else:
            #原来的代码
            #text = message.text
            text = rm_punctuation
        b.2.修改rasa/core/channels/channels.py line58（此位置是测试数据输入口，可对输入数据作对应修改，单但修改后的数据与events中数据一致）
          self.text = text.strip().strip('?').strip('!').strip('？').strip('！') if text else text
    6、slot_was_set：
        1、在domain.yml文件里先设置slot的键及其类型。
        2、用于story中插入词槽，slots可以充当键值存储。
        3、我们可以在action.py通过get_slot(“slot_key”)方法来获取slot的值，或者用SlotSet("slot_key","value")的方法来给slot设置值。
        4、槽的工作原理：故事(Stories)不会给你设置槽(slot)。槽必须在slot_was_set步骤之前由实体(entity)或自定义操作(custom action)设置。
        5、只有当前的词槽满足，story才能执行下面的会话。
        6、词槽不需要在句子中标注。词槽可以包括实体。

    7、entity：实体和插槽是差不多的。实体是需要在句子中标注的。
    注：entity需要标注，slot可以from_entity,from_intent,from_text，定义初始值等操作，类似于定义变量，可以在action中调用

    8、在story中使用user，可以模拟用户输入，执行“rasa test”可以对story进行测试
    例：
    - story: Happy path accepts suggestion  
      steps:
      - user: |
          hi
        intent: greet
      - action: utter_greet

    9、语料相关
    （1）同义词
    例:
    - intent: query_address
      examples: |
        - 有哪些[办事处]{"entity": "company", "value": "分公司"}?
    - synonym: 分公司
      examples: |
        - 办事处
    ===》“有哪些办事处？”，实体company:分公司
    （2）lookup（RegexFeaturizer，RegexEntityExtractor）
    例：
    - intent: query_address
      examples: |
        - [机票](ticket)在哪里订？
    - lookup: ticket
      examples: |
        - 飞机票
        - 火车票
    ===》“火车票在哪里订?”，也可以归为上一类，但无法识别“火车票”是ticker
    （3）regex（RegexFeaturizer，RegexEntityExtractor）
    - intent: query_address
      examples: |
        - [3](num)个香蕉
    - regex: num
      examples: |
        - [0-9一二三四五六七八九十]{1,}
    ===》“三十八个香蕉”，也可以归为上一类，识别“三十八”是num
    10、表单机器人
    （1）、表单实现步骤：
          激活表单-application_form-提交表单
          application_form
                 —— types:utter_ask_types/若比较复杂，添加actions:AskForSlotActiontypes/action_ask_types，给出提示用户输入types
                 validate_application_form——validate_types:验证用户输入的语句，获得types 
                 —— summarys:utter_ask_summarys/若比较复杂，添加actions:action_ask_summarys，给出提示用户输入summarys
                 ValidateclassForm/validate_application_form——validate_summarys:验证用户输入的语句，获得types ，若验证错误，提示用户重新输入
          action_slots_values:表单信息提取后，构建rasa返回语句给前端
          action_clear_application_form_slots：每次表单结束后清空词槽
     

    （2）、表单定义
    forms:
      application_form:
            types:
            # types从entity中获取
            - type: from_entity
              entity: types
              # 限制该词槽只能出现在[application,class,stop]中
              intent: [application,class,stop]
              # 限制不能从offscope意图中获取
              not_intent: offscope
              # 提取状态池
              conditions:
              - active_loop: application_form
                requested_slot: types
            summarys:
            # 表单值可以利用多种方法获取：from_entity、from_text、from_intent
            - type: from_entity
              entity: summarys
            # 无论用户输入什么，都会作为值
            - type: from_text
              # 防止输入stop的问题，自动填充到词槽中
              not_intent: stop
              conditions:
              - active_loop: application_form
                requested_slot: summarys
    （3）、from_text的值可能会匹配为form上面获得的类别，例如（2）中的types，继而覆盖原来的types值，解决方法：
    domain中新增一个backup的entity，在action_ask_summarys方法中，将types存入backup中，在validate_types中判断backup是否是none，若为none，可以存入新的，否则修改types的值为backup的值
    （4）、若用户表单外问表单详细内容，提示用户开始表单
    - rule: Example of an unhappy path
      steps:
      - intent: summary
      - action: action_stop
    （5）、在表单内若用户想停止重新开始，输入stop，清空已有form，提示用户重新开始表单
    - rule: stop form + continue
      steps:
        - intent: stop
        - action: action_stop
        - action: action_deactivate_loop
        - active_loop: null
    （6）、若前端用户在表单内长时间未响应，自动清空表单
    （7）、当用户问了一个问题，rasa陷入无法响应Circuit breaker tripped，最好在每个对话/form结束后，return [Restarted()]
#################################################################################################
【16、NLP github 项目】
#################################################################################################
1、文本对抗：github.com/Qdata/TextAttack
（1）文本扩充方法-准备需要扩充的数据example.csv
textattack augment --csv examples.csv --input-column text --recipe embedding --pct-words-to-swap .1 --transformations-per-example 3 --exclude-original --overwrite
（2）文本扩充方法-python可直接调用
    >>> from textattack.augmentation import EmbeddingAugmenter
    >>> augmenter = EmbeddingAugmenter()
    >>> s = 'What I cannot create, I do not understand.'
    >>> augmenter.augment(s)
注：目前测试结果中文效果不佳
2、文本对抗：github.com/williamSYSU/TextGAN-Pytorch
（详情见project/TextGAN-Pytorch）
(1)seqgan
适用于短文本
生成器训练
判别器训练
对抗训练
(2)leakgan
适用于长文本
训练方法：1、准备dataset/emnlp_news.txt和dataset/testdata/emnlp_news_test.txt
        2、修改对应run_seqgan.py或者run_leakgan.py文件中的训练参数
        3、python run/run_seqgan.py 2 0         (因为测试数据是实际数据，所以改成0 0，按实际情况修改)
        4、结果存在save目录下，ADV表示我们需要的对抗产生的文本，MLE表示生成七的最大相似估计数据
注：原代码只支持英文，为了满足中文也能测试的条件，这里测试的时候将utils/text_process.py中的get_tokenlized方法做了修改，例：
  def get_tokenlized(file):
      """tokenlize the file"""
      tokenlized = list()
      with open(file) as raw:
          for text in raw:
              #english corpus
              #text = nltk.word_tokenize(text.lower())
              #chinese corpus
              text = '/'.join(jieba.cut(text)).split("/")
              tokenlized.append(text)
      return tokenlized
3、扩充文本-github.com/makcedward/nlpaug
例：
>>> import nlpaug.augmenter.char as nac
>>> import nlpaug.augmenter.word as naw
>>> import nlpaug.augmenter.sentence as nas
>>> import nlpaug.flow as nafc

>>> from nlpaug.util import Action
>>> import os
>>> os.environ["MODEL_DIR"] = '../model'
>>> text = '我是实习生可以申请公司手机吗'
>>> # aug = nac.KeyboardAug()
>>> # augmented_text = aug.augment(text, n=2)
>>> ################################################
>>> aug = naw.ContextualWordEmbsAug(
>>>     model_path='bert-base-uncased', action="substitute")
>>> augmented_text = aug.augment(text)
>>> print("Original:")
>>> print(text)
注：目前测试结果中文效果不佳
#################################################################################################
【17、钉钉自定义机器人】
#################################################################################################
可以实现脚本控制钉钉定时发送消息
http://blog.csdn.net/qq_43605239/article/details/103822554?utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-4.baidujs&dist_request_id=1329188.20616.16179322504593479&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-4.baidujs
参考project/dingding.py
#################################################################################################
【18、http转成https】
#################################################################################################
1、申请域名
2、Ubuntu系统将域名指向指定IP
sudo vi /etc/hosts
例：35.231.145.151 gitlab.com
3、DNS
vi /etc/resolv.conf
sudo /etc/init.d/networking restart
4、certbot申请ssl证书
https://blog.csdn.net/leesmn/article/details/78084865
问题：
1、Trying to renew cert on nginx but getting “Problem binding to port 443: Could not bind to IPv4 or IPv6”
kill所有nginx进程或者
2、在执行步骤4之前先确保域名和ip都能ping通，且域名申请过备案，域名可以在网站中打开
#################################################################################################
【19、百度高级搜索】
#################################################################################################
intitle:  互联网发展的报告  filetype:pdf  inurl：com   2014..2019
#################################################################################################
【20、Django项目】
#################################################################################################
一、一些命令
#Django（mvt）
（1）新建django项目
django-admin startproject MyDjango
（2）创建app
cd MyDjango
python manage.py startapp MyBlog
（3）在settings中修改配置
INSTALLED_APPS加上MyBlog
（4）创建数据库
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
（5）管理数据库内容
在MyBlog中修改admin内容
（6）创建超级用户(root,root)
python manage.py createsuperuser
（7）运行
python manage.py runserver 0.0.0.0:9095
（8）管理视图Views（一个views对应一个url）
在MyBlog的views中添加视图控制
（9）在url中加上对应的view
url(r'^MyBlog/$', MyBlog_views.myBlog)
（9）管理模板配置Templates
在MyBlog文件夹里新建文件夹templates,然后在templates里新建BlogTemplate.html
（10）运行脚本创建测试数据:
python3 manage.py init_db
（11）demo.tmlsystem.com上的环境
运行python3 manage.py runserver 0.0.0.0:9095
则可以访问:
http://demo.tmlsystem.com:9095/
登录之后将到rest framework的界面。
二、环境安装
需要安装python3,django2等环节:
pip3 install -r requirements.txt
sudo pip3 install -e git+git://github.com/oscarmlage/django-cruds-adminlte/@0.0.17+git.081101d#egg=django-cruds-adminlte
到scripts目录下：
sudo sh fix_pymysql_for_django2.2.sh
#################################################################################################
【21、Excel公式】
#################################################################################################
$B$1表示：B1单元格绝对引用，下拉单元格不会变成B2、B3，C2、C3
$B1表示：固定B单元格，下拉单元格不会变成C1、C2，但可以编程B2、B3
#################################################################################################
【22、正反向代理nginx】
#################################################################################################
正向代理代理客户端，反向代理代理服务器
1、含义：通过nginx配置，浏览器可以用ip，ip:port，ip:port/test，ip:port/test.png查看服务
2、若部署nginx docker容器后，产生80端口占用的问题，需查看本地是否配置了nginx，且默认端口也为80，若是，可以修改本地nginx默认端口80为其他端口
3、修改nginx默认端口80，http://blog.csdn.net/YLD10/article/details/80242487
#################################################################################################
【23、web框架】
#################################################################################################
吞吐量(每秒处理的请求数最多)最高，用户平均等待时间最短
flask： 轻量级，主要是用来写接口的一个框架，实现前后端分离，提高开发效率。
diango：比较重量级的框架
fastapi：自动生成API文档
        1、解决跨域：allow_origins=origins或者['*']
        2、接口参数：
                方式1：class TextDada(BaseModel):
                        botname: str
                      async def rasa_bot_generation(request_data:TextDada):
                      请求参数：{'botname':'test'}
                      async def rasa_bot_generation(botname: str):
                      请求参数：botname='test'
                方式二：async def create_jira(files: List[UploadFile] = File(None),formDatas: str = Form(...)):
        3、接口参数形式：
                class FormData(BaseModel):
                    #大文件(前端上传的文件Uploadfile:binary),可以保存上传的文件
                    Uploadfile: UploadFile = File(...)
                    #多个大文件
                    Uploadfiles: List[UploadFile] = File(...)
                    #单个小文件(无法保存上传的文件)
                    file: bytes = File(...)
                    #多个小文件
                    files: List[bytes] = File(...)
                    # Formdata数据
                    formDatas: dict = Form(...)
                    # 字符串
                    botChname: str
                    # 默认为5009
                    botDeployPort:  Optional[str] = '5009'

sanic：异步编程工具箱，Sanic在处理长连接时特别有用
#################################################################################################
【24、网络-异步】
#################################################################################################
Async-await
同步指python服务，阻塞指python返回值
需要做一件事能不能立即得到返回应答，如果不能立即获得返回，需要等待，那就阻塞了，否则就可以理解为非阻塞
阻塞/同步：打一个电话一直到有人接为止
非阻塞/同步：打一个电话没人接，每隔10分钟再打一次，知道有人接为止
非阻塞/异步：打一个电话没人接，转到语音邮箱留言（注册），然后等待对方回电（call back),有返回机制
同步阻塞：你干吧，我看着你干
同步非阻塞：你干吧，我每隔5分钟来看看
异步阻塞：你干吧，好了告诉我，我等着
异步非阻塞：你干吧，好了告诉我，我先去忙别的了
#################################################################################################
【25、Spring Boot知识点】
#################################################################################################
1、sping boot各层之间的关系？
controller-->service-->serviceimpl-->dao-->daoimpl-->entity
controller业务控制层，负责前后端交互，接收数据和请求
service服务层，负责对接口的定义和逻辑处理
dao数据库交互层，负责对数据进行增删改查
entity实体层，负责定义数据类型
总结：个人理解DAO面向表，Service面向业务。后端开发时先数据库设计出所有表，然后对每一张表设计出DAO层，然后根据具体的业务逻辑进一步封装DAO层成一个Service层，对外提供成一个服务。
resource-->mapper（xml具体实现类）
2、测试
在test中编写测试类，鼠标指定对应类，右键运行该类
3、数据库客户端新建tables-->java-->generate.java代码生成器自动构建需要的service
4、maven打包可以取消test这块，否则maven打包会把所有单元测试跑一遍
#################################################################################################
【26、numpy与pandas】
#################################################################################################
numpy:
1、numpy是以矩阵为基础的数学计算模块，numpy更适合处理统一的数值数组数据，提供高性能的矩阵运算，数组结构为ndarray。
2、列表可以存储任意类型的数据，数组只能存储一种类型的数据，同时，数组提供了许多方便统计计算的功能（如平均值mean、标准差std等）。
1、创建数组
numpy.empty：创建指定形状、数据类型数组
numpy.empty([3,1],dtype=int):创建3*1的数组，数据类型是int
numpy.zeros:创建0数组
numpy.ones:创建1数组
2、指定数值范围创建数组
np.arange(0,6,1,dtype=int):生成0到6之间，步长为1的数组
np.linspace(1,10,8):生成1到10之间的8个等差数列
np.logspace(1,10,8):生成1到10之间的8个等比数列
3、切片索引
a=np.arange(10)
s=slice(2,7,2)#从索引2开始到索引7停止，间隔2
或
s=a[2:7:2]
或获取指定列和行的数字
a=np.arange([1,2,3],[3,4,5],[4,5,6])
a[...,1]#第二列元素
a[1,...]#第二行元素
a[...,1:]#第二列及剩下的所有元素
x[[0,1,2],  [0,1,0]]#获取数组(0,0)、(1,1)、(2，0)位置的数据，即1,4,4 

pandas:
1、pandas是基于numpy数组构建的，但二者最大的不同是pandas是专门为处理表格和混杂数据设计的，比较契合统计分析中的表结构。pandas数组结构有一维Series和二维DataFrame。
##########################################################################################


