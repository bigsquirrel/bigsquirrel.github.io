---
layout: post
title:  "Python 的简易通讯录"
date:   2015-04-14 12:00:00
tags:
    - Python
---
花了几个晚上自学 Python，在读完 A Byte of Python 后，关于书中给的一个小问题：
	**`创建你自己的命令行地址簿程序。在这个程序中,你可以添加、修改、删除和搜 索你的联系人(朋友、家人和同事等等)以及它们的信息(诸如电子邮件地址和/或 电话号码)。这些详细信息应该被保存下来以便以后提取。`**
做了一个简单的实现，贴代码：

	#!/usr/bin/python
	# Filename: address_book.py
	import string
	import pickle
	class People(object):
		def __init__(self, name='', tel='', email=''):
			self.name = name
			self.tel = tel
			self.email = email
		# def tostr(self):
		# 	return ('{0}|{1}|{2}'.format(self.name, self.tel, self.email))
		# def parse(self, pstr):
		# 	pstrs = pstr.split('|')
		# 	self.name = pstrs[0]
		# 	self.tel = pstrs[1]
		# 	self.email = pstrs[2]
		# 	return self

	def read_all():
		f = open('data', 'rb')
		try:
			return pickle.load(f)
		except EOFError:
			print('EOFError')
			pass
		except IndentationError:
			print('IndentationError')
			pass
		# while True:
		# 	line = f.readline()
		# 	if len(line) == 0:
		# 		break
		# 	p = People()
		# 	address[p.name] = p.parse(line)
		f.close()

	def view_all():
		for name, people in address.items():
			print('Contact: {0} {1} {2}'.format(people.name, people.tel, people.email))

	def search(name):
		if name in address:
			people = address[name]
			print('Contact: {0} {1} {2}'.format(people.name, people.tel, people.email))

	def modify(name):
		if name in address:
			people = address[name]
			people.tel = raw_input('Enter new tel:')
			people.email = raw_input('Enter new email:')
			address[name] = people
			view_all()
			save()

	def delete(name):
		if name in address:
			del address[name]
			save()

	def save():
		f = open('data', 'wb')
		pickle.dump(address, f)
		f.close()
		# for name, people in address.items():
		# 	f.write(people.tostr())
		# 	f.write('\r')


	address={}

	while True:
		address = read_all()
		print('1. View all')
		print('2. View sigle')
		print('3. Modify')
		print('4. Del')
		print('5. New')
		print('0. Exit')
		i = input('Enter operation:')
		if i == 1:
			view_all()
		elif i == 2:
			name = raw_input('Enter the name of contact you want to see:')
			search(name)
		elif i == 3:
			name = raw_input('Enter the name of contact you want to modify:')
			modify(name)
		elif i == 4:
			name = raw_input('Enter the name of contact you want to delete:')
			delete(name)
		elif i == 5:
			name = raw_input('Enter name:')
			tel = raw_input('Enter tel:')
			email = raw_input('Enter email:')
			p = People(name, tel, email)
			address[name] = p
			save()
		elif i == 0:
			print('Exit!')
			break;
		else:
			print('Input wrong!')
			continue

可以看到代码中我注释掉的部分是试图按照一定格式写入以及读取文件的，但实际过程中还是存在着 bug，后来阅读书中的提示**建一个类来表示人的信息。用字典来存储人的对象,可以将名字作为其键值。用 pickle 模块将对象永久地存在磁盘上。用字典的内置方法实现增加,删除和修改联系人的信息。**发现直接用`pickle`确实要方便的多。