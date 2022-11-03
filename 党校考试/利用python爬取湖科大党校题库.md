@[toc]

# 前言

* 入党积极分子/发展对象/预备党员培训虽说有题库，但是是做对一道显示一道的，有的不会的还得找半天，咱计算机的不能干这些没有技术含量的，因此写了个爬虫自动刷题，套出答案并记录在数据库中。
* 链接说明
  * 入党积极分子：https://dangxiao.hnust.edu.cn/index.php?s=/Exam/practice/lib/1
  * 发展对象：https://dangxiao.hnust.edu.cn/index.php?s=/Exam/practice/lib/2
  * 预备党员：https://dangxiao.hnust.edu.cn/index.php?s=/Exam/practice/lib/3


# 主要功能

* 爬取题库，将题目和选项以及正确答案记录在mysql中
* 去除重复的题目
* 记录题目被爬到的次数，统计频率，找出出得多的高频题目（这个可能没什么用）

# 注意事项

1. 题库是一个整体，没有严格区分，考试题目是交错的，每个阶段抽取的题目都可能抽取到其他阶段的，这个只是把整个题库搞下来了，具体哪个阶段偏好抽哪些题是不清楚的，所以拿题库背题是不明智的（太多了）。那怎么办呢？hh自己想办法吧^_~
2. 要更改的内容主要是要改数据库信息，接下来就可以跑程序了。

2. 数据库结构为：（DDL）记得建数据库！！！

```sql
-- auto-generated definition
create table dangxiaotiku_fzdx
(
    problem_id int           not null
        primary key,
    title      varchar(500)  not null,
    A          varchar(100)  null,
    B          varchar(100)  null,
    C          varchar(100)  null,
    D          varchar(100)  null,
    E          varchar(100)  null,
    answer     varchar(20)   null,
    count      int default 1 not null,
    constraint dangxiaotiku_fzdx_problem_id_uindex
        unique (problem_id)
);
```
* 示例
![部分数据库数据](https://img-blog.csdnimg.cn/9d9b4425c0cb432489173ac882797499.png)

# 代码

```python
from time import sleep
import pymysql
import requests
from lxml import html

requests.packages.urllib3.disable_warnings()


def connect_database():
	# 打开数据库连接（要改成自己的）
	db = pymysql.connect(host="localhost", port=3306, user="root", password="root", database="mydb")
	# 使用cursor()方法获取操作游标
	cursor = db.cursor()
	return db, cursor


# 在数据库中查询，看看题目是否已经有了
# 如果在数据库中存在，给该题目出现的次数加一
def has_exists(problem_id):
	db = ''
	cursor = ''
	try:
		db, cursor = connect_database()
		# 执行sql语句，返回在数据库中的记录数
		count = cursor.execute('select count from mydb.dangxiaotiku_fzdx where problem_id=' + str(problem_id))
		if count == 0:
			return False
		else:
			t = cursor.fetchall()
			print(t)
			data = int(t[0][0]) + 1
			cursor.execute(
				'update mydb.dangxiaotiku_fzdx set count=' + str(data) + ' where problem_id=' + str(problem_id))
			db.commit()
			return True
	except Exception as e:
		print(e)
		db.close()
	finally:
		# 关闭数据库连接
		cursor.close()
		db.close()


# 写入发展对象题库数据库
def write_fzdx(problem_id, title, answer_content_list, right_answer):
	problem_id = problem_id.split('_')[1].replace('[]', '')
	print('开始保存题目：', problem_id)
	a = ['', '', '', '', '']
	i = 0
	# print(answer_content_list.__len__())
	while i < answer_content_list.__len__():
		a[i] = answer_content_list[i]
		i = i + 1

	if type(right_answer) is list:
		print('list数组')
		right_answer = ','.join(right_answer)

	print(right_answer)
	# SQL 修改数据
	insert_sql = "insert into mydb.dangxiaotiku_fzdx(problem_id, title, A, B, C, D, E, answer) values(" \
	             + problem_id + \
	             ",'" + title + \
	             "','" + a[0] + \
	             "','" + a[1] + \
	             "','" + a[2] + \
	             "','" + a[3] + \
	             "','" + a[4] + \
	             "','" + right_answer + "')"

	db = ''
	cursor = ''
	try:
		db, cursor = connect_database()
		# 执行sql语句
		cursor.execute(insert_sql)
		# 提交到数据库执行
		db.commit()
		print(problem_id, '保存完毕')
	except Exception as e:
		print(e)
		# 如果发生错误则回滚
		db.rollback()
		print(problem_id, '保存失败')
	finally:
		# 关闭数据库连接
		cursor.close()
		db.close()


# 发送请求，返回正确答案
def judge(number, c):
	# 提交答案（post）
	submit_url = 'https://dangxiao.hnust.edu.cn/index.php?s=/exam/practice'
	data = {
		# 题目编号，如problem_914[]或problem_914
		number: c,
		'method': 'submit'
	}
	headers = {
		'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.198 Safari/537.36',
		'Host': 'dangxiao.hnust.edu.cn',
		'Origin': 'https://dangxiao.hnust.edu.cn',
		'Referer': 'https://dangxiao.hnust.edu.cn/index.php?s=/Exam/practice/lib/2'
	}
	print(data)
	response = requests.post(url=submit_url, headers=headers, data=data, verify=False)
	json = response.json()
	# 状态码 响应为200，正确，有响应，status=100 \u7b54\u6848\u4e0d\u6b63\u786e 为错误
	print(json)
	if json.__len__() > 0:
		status = json.get('status')
		print(status)
		if status == 100:
			print('答案错误')
			return 'error'
		elif status == 200:
			print('答案正确，为：', c)
			return c
	else:
		print('ERROR')
		exit(1)


def get_one(type):
	# 获取题目（get）
	# 入党积极分子1 发展对象2 预备党员3（均不完全重合，如要爬取完整题库，三个都要爬）
	target_url = 'https://dangxiao.hnust.edu.cn/index.php?s=/Exam/practice/lib/' + str(type)
	# 请求方法: POST
	headers = {
		'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.198 Safari/537.36',
		'Referer': 'https://dangxiao.hnust.edu.cn/index.php?s=/exam/index'
	}

	# 因为https是第三方CA证书认证的
	# 解决办法 是：告诉web 忽略证书访问 verify = False
	response = requests.get(url=target_url, headers=headers, verify=False, timeout=10)

	resp = response.text
	# print(resp)
	tree = html.etree.HTML(resp)
	# 题目编号
	number = tree.xpath('//*[@id="form1"]/table/tbody/tr[1]/td[1]/input/@name')[0]
	print(number)

	# 获取纯id
	# 在数据库中不存在才寻找答案并插入
	if not has_exists(number.split('_')[1].replace('[]', '')):
		# 题目标题
		title = tree.xpath('//*[@id="form1"]/table/thead/tr/th/text()')[0].replace(' ', '')
		print(title)

		# 判断有几个选项
		trs = tree.xpath('//*[@id="form1"]/table/tbody/tr')
		answer_count = trs.__len__()
		print(answer_count)

		answer_content_list = []
		# 获取选项内容
		for tr in trs:
			answer_content = tr.xpath('./td[2]/text()')[0]
			answer_content_list.append(answer_content)
		print(answer_content_list)

		# 多选题
		if number.find('[]') != -1:
			print('多选题')
			c_list = [['A', 'B'], ['A', 'C'], ['A', 'D'], ['A', 'E'], ['B', 'C'], ['B', 'D'], ['B', 'E'], ['C', 'D'],
			          ['C', 'E'],
			          ['D', 'E'],
			          ['A', 'B', 'C'], ['A', 'B', 'D'], ['A', 'B', 'E'], ['B', 'C', 'D'], ['B', 'C', 'E'],
			          ['B', 'D', 'E'], ['A', 'C', 'D'], ['A', 'C', 'E'], ['A', 'D', 'E'], ['C', 'D', 'E'],
			          ['A', 'B', 'C', 'D'], ['A', 'B', 'C', 'E'], ['A', 'B', 'D', 'E'], ['B', 'C', 'D', 'E'],
			          ['A', 'C', 'D', 'E'],
			          ['A', 'B', 'C', 'D', 'E']]
			i = 0
			while i < c_list.__len__():
				# 抽取一种可能的答案
				c = c_list[i]
				# 判断返回值
				t = judge(number, c)
				if 'error' == t:
					i = i + 1
				else:
					write_fzdx(number, title, answer_content_list, t)
					break
		else:
			print('单选题')
			# 单选题
			i = 1
			# 设置需要提交的选项
			while i <= answer_count:
				c = ''
				if i == 1:
					c = 'A'
				elif i == 2:
					c = 'B'
				elif i == 3:
					c = 'C'
				elif i == 4:
					c = 'D'
				elif i == 5:
					c = 'E'
				else:
					print('ERROR')
					exit(1)

				# 判断返回值
				t = judge(number, c)
				if 'error' == t:
					i = i + 1
				else:
					write_fzdx(number, title, answer_content_list, t)
					break
				sleep(0.5)


if __name__ == '__main__':
	i = 0
	# 爬10000次
	cishu = 10000
	# 爬全部题库（不过好像预备党员的题目发展对象包括的非常多）
	while i < cishu:
		if i % 3 == 0:
			get_one(1)
		elif i % 3 == 1:
			get_one(2)
		else:
			get_one(3)
		i = i + 1
		sleep(1)
```