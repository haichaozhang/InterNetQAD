# 收集Avrix论文的总结

## 概述
> &ensp;&ensp;整个项目由数据采集（Python）,数据存储（Mysql），数据可视化（C#）组成。  
> &ensp;&ensp;数据采集主要负责从网络上，获取Avrix的论文基本信息与论文下载地址。将其存储至MySQL，此过程中将分析Avrix上网页的结构，依靠依然是Chrome进行。  
> &ensp;&ensp;数据存储食用的MySQL,其实我蛮想用SqlServer的，之前的东家是SqlServer，操作极其稳定与简单，速度很快，功能很全。新东家节约成本给我们多发点工资，用起来MySql了，使用了一段时间，大体上差不多，数据转存备份比转存比SqlServer复杂一些。  
> &ensp;&ensp;数据可视化，目前做的部分是C#编译的一个查询界面，可以查询到相应的论文，可以从数据库中检索出论文，并链接指向的PDF文件，使用系统默认软件打开PDF。
## 分项陈述
### pyhon数据采集部分
> &ensp; 直接上代码

``` Python
# -*- coding:utf-8 -*-

#作者：Qt.chao
#时间：2018/02/20
#综述：从Cornell University的图书馆Arvix主页上，获取计算机领域Computer Vision and Pattern Recognition的相关论文

import urllib.request
import pymysql
from bs4 import BeautifulSoup
import requests
import time
import re
import os

# 数据库连接基础类
class Conn_Mssql:
    #查询Mysql使用sql语句
    def Select_mssql(strsql):
        #数据库连接信息
        conn = pymysql.connect("localhost", "**username**", "**password**", "internetdaq", charset="utf8")
        cur = conn.cursor()
        cur.execute(strsql)
        return cur
    #插入与更新sql语句使用
    def InsertOrUpdate_mssql(strsql):
        # 数据库连接信息
        conn = pymysql.connect("localhost", "**username**", "**password**", "internetdaq", charset="utf8")
        cur = conn.cursor()
        cur.execute(strsql)
        conn.commit()
        conn.close()
        return cur


#获取网络信息中的信息，并存储
class Get_HttpMessage:
    # 下载文件函数（根据连接地址，下载至D:StorePDF目录下）
    def getFile(url):
        try:
            file_name = url.split('/')[-1]
            file_path = "StorePDF\\"+file_name
            u = urllib.request.urlopen(url)
        except :
            print(url, "url file not found")
            return
        block_sz = 90192
        with open(file_path, 'wb') as f:
            while True:
                buffer = u.read(block_sz)
                if buffer:
                    f.write(buffer)
                else:
                    break
        # 成功获取下载并打印下载信息
        print("Sucessful to download" + " " + file_name)
    # 获取文章中的PDF文档链接地址并下载
    def getPaperFile(url,file_name,path):
        try:
            file_name = url.split('/')[-1]
            file_path = path +"\\"+file_name
            u = urllib.request.urlopen(url)
        except :
            print(url, "url file not found")
            return
        block_sz = 901920
        with open(file_path, 'wb') as f:
            while True:
                buffer = u.read(block_sz)
                if buffer:
                    f.write(buffer)
                else:
                    break
        print("Sucessful to download" + " " + file_name)

    # 从页面中获取论文数据（PDF下载地址与论文的标题等信息）
    def startGet(strUrl):
        print('start')
        # 链接的APPM网络
        url = strUrl
        request = urllib.request.Request(url)
        response = urllib.request.urlopen(request)
        data = response.read()
        soup = BeautifulSoup(data, "lxml")
        for link1 in soup.find_all(id=re.compile("dlpage")):
            for linklist in link1.find_all("dl"):
                # 论文连接地址相关的信息
                linklistLpdf = linklist.find_all("dt")
                # 论文标题作者等相关信息
                linklistLName = linklist.find_all("dd")

                # 节点信息长度
                cont_pdf = len(linklistLpdf)
                cont_Name = len(linklistLName)

                if cont_pdf == cont_Name :
                    for linkNum in range(0,(cont_Name)):
                        onepdf = linklistLpdf[linkNum].find_all(href=re.compile("pdf"))

                        if len(onepdf)>0 :
                            # PDF下载的连接地址
                            pdfurl ="https://arxiv.org"+ (onepdf[0])['href']+".pdf"

                            # 文章的详细信息链接地址
                            paper_DetailUrls = linklistLpdf[linkNum].find_all(href=re.compile("abs"))
                            paper_DetailUrl = "https://arxiv.org" + (paper_DetailUrls[0])['href']

                            # 包含论文题目，作者，摘要的节点
                            oneNames = linklistLName[linkNum].find_all(class_="meta")

                            # Paper的编号信息
                            oneName = paper_DetailUrls[0].get_text().replace("'","^",999)

                            # 文章的标题
                            Paper_Titles = oneNames[0].find_all(class_="list-title mathjax")
                            Paper_Title = Paper_Titles[0].get_text().replace("'","^",999)

                            # 文章的作者
                            Paper_Authors = oneNames[0].find_all(class_="list-authors")
                            Paper_Author =""
                            for Paper_Authorlist in Paper_Authors[0].find_all("a"):
                                Paper_Author =Paper_Author +Paper_Authorlist.get_text()+"|"
                            Paper_Author = Paper_Author.replace("'","^",999)
                            Paper_Author = Paper_Author[:-1]

                            # 获取摘要信息
                            time.sleep(2)
                            request2 = urllib.request.Request(paper_DetailUrl)
                            response2 = urllib.request.urlopen(request2)
                            data2 = response2.read()
                            soup2= BeautifulSoup(data2, "lxml")
                            Paper_ABSTRACTS = soup2.find_all(class_="abstract mathjax")
                            if len(Paper_ABSTRACTS)>0 :
                                Paper_ABSTRACT = Paper_ABSTRACTS[0].get_text().replace("'", "^", 999)
                            else:
                                Paper_ABSTRACT = ""
                            # 分类信息
                            PAPER_SUBJECT = ""
                            PAPER_SUBJECTs = soup2.find_all(class_="tablecell subjects")
                            if len(PAPER_SUBJECTs)>0 :
                                PAPER_SUBJECT = PAPER_SUBJECTs[0].get_text().replace("'", "^", 999)
                            else:
                                PAPER_SUBJECT = ""

                            # 获取网页的详细内容信息
                            Paper_Detail = soup2.prettify().replace("'","^",999)
                            Paper_Detail = Paper_Detail.replace("-->","  -->",999)

                            strSQL= "CALL SaveTheCornellUniversityPaper(0,'"+oneName+"','" + Paper_Detail + "','"+pdfurl+"',0,'" + Paper_Title + "','" + Paper_Author + "','" + Paper_ABSTRACT + "','" + PAPER_SUBJECT + "')"
                            strSQL = strSQL.encode('utf8')
                            try:
                                # 存储地址信息
                                Conn_Mssql.InsertOrUpdate_mssql(strSQL)
                                # Get_HttpMessage.getFile(pdfurl)
                                time.sleep(0.5)
                                print('母页面MySQL存储成功')
                            except:
                                print(strSQL)
                                print('母页面MySQL存储失败')

                            # time.sleep(1)
    # 从主界面获取主界面与分界面的连接信息
    def selectUrl(strUrl):
        print(strUrl)
        Get_HttpMessage.startGet(strUrl)
        request = urllib.request.Request(strUrl)
        response = urllib.request.urlopen(request)
        data = response.read()
        soup = BeautifulSoup(data, "lxml")
        for link1 in soup.find_all(id=re.compile("dlpage")):
            linklists = link1.find_all("small")
            for linkUrl in linklists[0].find_all("a"):
                oneUrl = "https://arxiv.org" + linkUrl["href"]
                print(oneUrl)
                Get_HttpMessage.startGet(oneUrl)
    # 从数据库中获取链接地址，下载论文
    def DownlooadPDFFromMysql(str= ""):
        urlRows = Conn_Mssql.Select_mssql("SELECT TILE_NAME ,URL from cornell_paper WHERE FLAGE = 0 order BY UID DESC  ;")
        for urlRow in urlRows:
            try:
                Get_HttpMessage.getPaperFile(urlRow[1],urlRow[0]+".pdf","D:\\CornellLibrary")
                urlRowfiles =urlRow[0].split(":")
                urlRowfile = urlRowfiles[1]
                charFILE_PATH = "D:\\\\CornellLibrary\\\\"+urlRowfile+".pdf"
                strsql = "UPDATE cornell_paper SET FLAGE = 1 , FILE_PATH = '"+ charFILE_PATH +"' WHERE URL = '"+urlRow[1]+"'"
                
                Conn_Mssql.InsertOrUpdate_mssql(strsql)
                print("下载成功"+ charFILE_PATH )
            except:
                print("下载失败")
            finally:
                time.sleep(3)
# 程序入口
# Arvix相关主页的链接地址
MainUrl = '网络地址'
#  从网页中抓取相关数据信息
Get_HttpMessage.selectUrl(MainUrl)
# 从MySQL数据库中下载论文
Get_HttpMessage.DownlooadPDFFromMysql()

```
> 如上即我采集Arvix主页上[Computer Vision and Pattern Recognition](https://arxiv.org/list/cs.CV/recent)相关信息信相关信息信，先将网页信息逐个解析，分类存储。将网页的信息存储后，从数据库中，按照标志位，根据下载地址信息，下载相应论文，具体流程如图所示。
```
   graph TB  
  A[Arvix主页信息获取] --> B(获取分页信息)
  B--> C(分标题存储至MySQL) 
  C--> D(根据地址下载论文)

```

### MySQL数据存储部分
> 在整个采集的过程中，是将数据存储在一数据库的一张表中，数据库名称为internetdaq，表（cornell_paper）的结构是：

| 字段说明 | 字段名 |  字段类型|
| --------   | -----:  | -----: |
| 自增无重复编号  | UID |  bigint(20)|
| 保存时间        |   SAVE_TIME   |  datetime|
| 标题名称        |    TILE_NAME  | varchar(200)|
| 存储类型        |    TYPE  | int(11)|
| 网页内详细信息        |   PAGE_DETAIL | text|
| 标志位        |   FLAGE | int(11) |
| 文章下载连接地址        |   URL |  varchar(255) |
| 文章题目        |   PAPER_TITLE |  varchar(1500) |
| 文章作者        |   PAPER_AUTHOR |  varchar(500)  |
| 文章摘要        |   PAPER_ABSTRACT |  varchar(6000) |
| 文章分类        |   PAPER_SUBJECT |  varchar(500) |
| 文件路径        |   FILE_PATH |  varchar(300) |


> 数据库表结构SQL语句为：
```SQL
DROP TABLE IF EXISTS `cornell_paper`;
CREATE TABLE `cornell_paper` (
  `UID` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '自增无重复编号',
  `SAVE_TIME` datetime DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP COMMENT '保存时间',
  `TILE_NAME` varchar(200) COLLATE utf8_unicode_ci DEFAULT NULL COMMENT '标题名称',
  `TYPE` int(11) DEFAULT NULL COMMENT '存储类型',
  `PAGE_DETAIL` text COLLATE utf8_unicode_ci COMMENT '网页内详细信息',
  `FLAGE` int(11) DEFAULT NULL COMMENT '标志位',
  `URL` varchar(255) COLLATE utf8_unicode_ci DEFAULT NULL COMMENT '文章下载连接地址',
  `PAPER_TITLE` varchar(1500) COLLATE utf8_unicode_ci DEFAULT NULL COMMENT '文章题目',
  `PAPER_AUTHOR` varchar(500) COLLATE utf8_unicode_ci DEFAULT NULL COMMENT '文章作者',
  `PAPER_ABSTRACT` varchar(6000) COLLATE utf8_unicode_ci DEFAULT NULL COMMENT '文章摘要',
  `PAPER_SUBJECT` varchar(500) COLLATE utf8_unicode_ci DEFAULT NULL COMMENT '文章分类',
  `FILE_PATH` varchar(300) COLLATE utf8_unicode_ci DEFAULT NULL COMMENT '文件路径',
  PRIMARY KEY (`UID`)
) ENGINE=InnoDB AUTO_INCREMENT=2445 DEFAULT CHARSET=utf8 COLLATE=utf8_unicode_ci;

```

> 数据库使用的存储结构SQL语句为：

```SQL
DROP PROCEDURE IF EXISTS `SaveTheCornellUniversityPaper`;
CREATE PROCEDURE SaveTheCornellUniversityPaper(in int_Type int, in CharTILE_NAME varchar(200),
in CharPAGE_DETAIL text,in CharURL varchar(254),in int_Flage int
,in CharPAPER_TITLE varchar(1500)
,in CharPAPER_AUTHOR varchar(500)
,in CharPAPER_ABSTRACT varchar(6000)
,in CharPAPER_SUBJECT varchar(500)
)
MODIFIES SQL DATA
COMMENT'从Cornell大学的论文主页获取论文的信息并存储 int_Type(类型) CharTILE_NAME(标题) CharPAGE_DETAIL(网页信息) CharURL(链接地址) int_Flage(标志位)'
BEGIN
    DECLARE pdfCount int DEFAULT 0;
  SELECT COUNT(URL) INTO pdfCount from cornell_paper WHERE URL = CharURL;
IF pdfCount< 1 THEN
    INSERT INTO cornell_paper
      (SAVE_TIME,TYPE,TILE_NAME,PAGE_DETAIL,FLAGE,URL,PAPER_TITLE,PAPER_AUTHOR,PAPER_ABSTRACT,PAPER_SUBJECT)
       VALUES
       (NOW(),int_Type,CharTILE_NAME,CharPAGE_DETAIL,int_Flage,CharURL,CharPAPER_TITLE,CharPAPER_AUTHOR,CharPAPER_ABSTRACT,CharPAPER_SUBJECT);
END IF;

END

```

### CSharp数据查询部分
> 此部分主要是用C#从数据库中提取数据，并查询连接查看，主要是为了平时查看论文的具体内容和PDF文档，用于实现的代码也较为简单，没做过多修饰，在接下完善中，可能使用python或C#进行进一步的语义分析与数据可视化，目前主要逻辑如何下所示

#### 主查询界面
```C#
using System;
using System.Collections.Generic;
using System.Data;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Data;
using System.Windows.Documents;
using System.Windows.Input;
using System.Windows.Media;
using System.Windows.Media.Imaging;
using System.Windows.Navigation;
using System.Windows.Shapes;
using LCS;

namespace PGCL
{
    /// <summary>
    /// MainWindow.xaml 的交互逻辑
    /// </summary>
    public partial class MainWindow : Window
    {
        public MainWindow()
        {
            InitializeComponent();
        }

        private DataTable dataGridDataTable;

        /// <summary>
        /// 窗体登录
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="e"></param>
        private void MainWindow_OnLoaded(object sender, RoutedEventArgs e)
        {
            
        }

        /// <summary>
        /// 刷新按钮
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="e"></param>
        private void BtnUpdate_OnClick(object sender, RoutedEventArgs e)
        {
            ShowAllDate();
        }

        /// <summary>
        /// 刷新显示所有数据列表
        /// </summary>
        private void ShowAllDate()
        {
            DataTable myDataTable = LCS_Lib_DataMySql.Mysql_SelectDataTable("SELECT SAVE_TIME,PAPER_TITLE,PAPER_SUBJECT,URL FROM `cornell_paper` ORDER BY UID DESC ;");

            dataGridDataTable = new DataTable();

            dataGridDataTable.Columns.Add("Num", typeof(string));
            dataGridDataTable.Columns.Add("Save_time", typeof(string));
            dataGridDataTable.Columns.Add("Title", typeof(string));
            dataGridDataTable.Columns.Add("Subject", typeof(string));
            dataGridDataTable.Columns.Add("URL", typeof(string));
            int IntNum = 1;

            foreach (DataRow VARDataRow in myDataTable.Rows)
            {
                DataRow oneDataRow = dataGridDataTable.NewRow();
                oneDataRow["Num"] = IntNum.ToString();
                oneDataRow["Save_time"] = VARDataRow["SAVE_TIME"].ToString();
                oneDataRow["Title"] = VARDataRow["PAPER_TITLE"].ToString();
                oneDataRow["Subject"] = VARDataRow["PAPER_SUBJECT"].ToString();
                oneDataRow["URL"] = VARDataRow["URL"].ToString();
                dataGridDataTable.Rows.Add(oneDataRow);
                IntNum++;
            }

            Paper_datagrid.ItemsSource = dataGridDataTable.DefaultView;


        }
        /// <summary>
        /// 双击表格
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="e"></param>
        private void Paper_datagrid_OnMouseDoubleClick(object sender, MouseButtonEventArgs e)
        {
            DataRowView onreRowView = (DataRowView) Paper_datagrid.SelectedItem;
            DataRow oneDataRow = onreRowView.Row;

            DataTable myDataTable = LCS_Lib_DataMySql.Mysql_SelectDataTable("SELECT PAPER_TITLE,PAPER_AUTHOR,PAPER_SUBJECT,PAGE_DETAIL,FILE_PATH" +
                                                                            " FROM cornell_paper WHERE URL = '" + oneDataRow["URL"].ToString() + "' ");
            winDetail myWinDetail = new winDetail();
            myWinDetail.strTile = myDataTable.Rows[0]["PAPER_TITLE"].ToString();
            myWinDetail.strAuthor = myDataTable.Rows[0]["PAPER_AUTHOR"].ToString();
            myWinDetail.strSubject = myDataTable.Rows[0]["PAPER_SUBJECT"].ToString();
            myWinDetail.strpage_detail = myDataTable.Rows[0]["PAGE_DETAIL"].ToString();
            string[] arelinkStrings = myDataTable.Rows[0]["FILE_PATH"].ToString().Split(':');
            string strLkink = "";
            if (myDataTable.Rows[0]["FILE_PATH"].ToString().Length>0)
            {
                strLkink = myDataTable.Rows[0]["FILE_PATH"].ToString();
            }
            else
            {
                strLkink = "";
            }
            
            myWinDetail.strPDFlink = strLkink;
            myWinDetail.ShowDialog();
        }
    }
}


```

#### 详细信息查询
```C#
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Reflection;
using System.Text;
using System.Threading.Tasks;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Data;
using System.Windows.Documents;
using System.Windows.Input;
using System.Windows.Media;
using System.Windows.Media.Imaging;
using System.Windows.Navigation;
using System.Windows.Shapes;

namespace PGCL
{
    /// <summary>
    /// winDetail.xaml 的交互逻辑
    /// </summary>
    public partial class winDetail : Window
    {
        public winDetail()
        {
            InitializeComponent();
        }
        /// <summary>
        /// 题目
        /// </summary>
        public string strTile { set; get; }

        /// <summary>
        /// 作者
        /// </summary>
        public string strAuthor { set; get; }

        /// <summary>
        /// 分类标签
        /// </summary>
        public string strSubject { set; get; }

        /// <summary>
        /// 网页详细信息
        /// </summary>
        public string strpage_detail { set; get; }

        /// <summary>
        /// PDF连接地址
        /// </summary>
        public string strPDFlink { set; get; }

        /// <summary>
        /// 关闭按钮
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="e"></param>
        private void Btn_close_OnClick(object sender, RoutedEventArgs e)
        {
            this.Close();
        }

        /// <summary>
        /// 窗体登录
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="e"></param>
        private void WinDetail_OnLoaded(object sender, RoutedEventArgs e)
        {
            lab_tile.Content = strTile;
            lab_author.Content = strAuthor;
            lab_subject.Content = strSubject;
            
            web_detail.NavigateToString(strpage_detail);
            
            lab_link.Content = strPDFlink;
            lab_link.Foreground = new SolidColorBrush(Colors.Blue);
        }

        public void SuppressScriptErrors(WebBrowser wb, bool Hide)
        {
            FieldInfo fiComWebBrowser = typeof(WebBrowser).GetField("_axIWebBrowser2", BindingFlags.Instance | BindingFlags.NonPublic);
            if (fiComWebBrowser == null) return;

            object objComWebBrowser = fiComWebBrowser.GetValue(wb);
            if (objComWebBrowser == null) return;

            objComWebBrowser.GetType().InvokeMember("Silent", BindingFlags.SetProperty, null, objComWebBrowser, new object[] { Hide });
        }
        
        private void Web_detail_OnNavigated(object sender, NavigationEventArgs e)
        {
            SuppressScriptErrors(web_detail, true);
        }
        /// <summary>
        /// 双击表格
        /// </summary>
        /// <param name="sender"></param>
        /// <param name="e"></param>
        private void Lab_link_OnMouseDoubleClick(object sender, MouseButtonEventArgs e)
        {
            Process MyProcess = new Process();
            MyProcess.StartInfo.FileName = lab_link.Content.ToString();
            MyProcess.StartInfo.Verb = "Open";
            MyProcess.StartInfo.CreateNoWindow = true;
            MyProcess.Start();
        }
    }
}

```

## 总结
> 上述即为我这次数据采集的实践，再接下来的学习中，我会向着词频分析与数据可视化方向进一步拓展。上述代码不能直接复制粘贴使用，仅供参考。

