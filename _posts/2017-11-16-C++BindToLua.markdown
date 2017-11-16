---
layout: post
title: C++对象绑定到lua
date: 2017-11-16
---

### lua.hpp
    // lua.hpp  
	// Lua header files for C++  
	// <<extern "C">> not supplied automatically because Lua also compiles as C++  
	  
	#ifdef __cplusplus    
	extern "C" {  
	#endif  
	    #include "lua.h"  
	    #include "lualib.h"  
	    #include "lauxlib.h"  
	#ifdef __cplusplus  
	}  
	#endif  

### Student.h
	#ifndef __STUDENT_H_  
	#define __STUDENT_H_  
	  
	  
	#include "lua.hpp"  
	  
	class Student  
	{  
		public:  
		    Student();  
		    void SetAge(int nAge);  
		    int GetAge();  
		  
		    int m_nAge;  
	};  
	  
	  
	#ifdef __cplusplus  
	extern "C" {  
	#endif  
	    int luaopen_student(lua_State *L);  
	#ifdef __cplusplus  
	}  
	#endif  
	  
	#endif 

### Student.cpp
	#include "Student.h"  
	  
	Student::Student()  
	{  
	    m_nAge = 0;  
	}  
	  
	void Student::SetAge(int nAge)  
	{  
	    this->m_nAge = nAge;  
	}  
	  
	  
	int Student::GetAge()  
	{  
	    return this->m_nAge;  
	}  
	  
	static int Create(lua_State *L) {  
	    Student **s =  (Student**)lua_newuserdata(L, sizeof(Student*));  
	    *s = new Student;  
	  
	    // 将userdata自定义数据与元表Meta_Student相关联  
	    luaL_getmetatable(L, "Meta_Student");  
	    lua_setmetatable(L, -2);  
	  
	    return 1;  
	}  
	  
	  
	static int SetAge(lua_State *L) {  
	    Student **s = (Student**)luaL_checkudata(L, 1, "Meta_Student");  
	    luaL_argcheck(L, s != NULL, 1, "invalid user data");  
	  
	    int nAge = (int)lua_tonumber(L, -1);  
	    (*s)->SetAge(nAge);  
	  
	    return 0;  
	}  
	  
	static int GetAge(lua_State *L) {  
	    Student **s = (Student**)luaL_checkudata(L, 1, "Meta_Student");  
	    luaL_argcheck(L, s != NULL, 1, "invalid user data");  
	  
	    lua_pushnumber( L, (lua_Number)((*s)->GetAge()) );  
	  
	    return 1;  
	}  
	  
	static int AutoDel(lua_State *L) {  
	    Student **s = (Student**)luaL_checkudata(L, 1, "Meta_Student");  
	    luaL_argcheck(L, s != NULL, 1, "invalid user data");  
	  
	    delete (*s);  
	  
	    return 0;  
	}  
	  
	  
	int luaopen_student(lua_State *L)   
	{  
	    luaL_Reg reg[] =   
	    {  
	        {"Create", Create },  
	        {NULL,NULL},  
	    };  
	  
	    // 创建名字为“Meta_Student”元表  
	    luaL_newmetatable(L, "Meta_Student");  
	  
	    //修改元表“Meta_Student”查找索引，把它指向“Meta_Student”自身  
	    lua_pushvalue(L, -1);  
	    lua_setfield(L, -2, "__index");  
	  
	    // GetAge方法  
	    lua_pushcfunction(L, GetAge);  
	    lua_setfield(L, -2, "GetAge");  
	  
	    // SetAge方法  
	    lua_pushcfunction(L, SetAge);  
	    lua_setfield(L, -2, "SetAge");  
	  
	    // 自动删除，如果表里有__gc,Lua的垃圾回收机制会调用它。  
	    lua_pushcfunction(L, AutoDel);  
	    lua_setfield(L,-2,"__gc");  
	  
	    // 注册reg中的方法到Lua中（“Student”相当于类名）,reg中的方法相当于Student成员函数  
	    luaL_register(L, "Student", reg);  
	  
	    return 1;  
	}

### main.lua  
	print(Student)    
	--[[   
	    1.打印Student地址  
	    2.打印nil --> luaL_register(L, "Student", reg); 这段代码没有执行成功  
	]]  
	  
	local p = Student.Create()  
	--[[  
	调用C++代码：  
	static int Create(lua_State *L) {  
	    Student **s =  (Student**)lua_newuserdata(L, sizeof(Student*));  
	    *s = new Student;  
	  
	    // 将userdata自定义数据与元表Meta_Student相关联  
	    luaL_getmetatable(L, "Meta_Student");  
	    lua_setmetatable(L, -2);  
	  
	    return 1;  
	}  
	  
	@retrun p-> Student*  
	]]  
	  
	p:SetAge(1200)     -- 调用方法1  
	p.SetAge(p, 1000)   -- 调用方法2  
	--[[  
	p:SetAge() p(userdata)没有注册SetAge方法, 就会寻找与p(userdata)相关联的元表Meta_Student有SetAge方法！并调用！  
	  
	调用C++代码：  
	static int SetAge(lua_State *L) {  
	    Student **s = (Student**)luaL_checkudata(L, 1, "Meta_Student");  
	    luaL_argcheck(L, s != NULL, 1, "invalid user data");  
	  
	    int nAge = (int)lua_tonumber(L, -1);  
	    (*s)->SetAge(nAge);  
	  
	    return 0;  
	}  
	  
	]]  
	  
	  
	print(p:GetAge())  
	--[[  
	调用C++代码：  
	static int GetAge(lua_State *L) {  
	    Student **s = (Student**)luaL_checkudata(L, 1, "Meta_Student");  
	    luaL_argcheck(L, s != NULL, 1, "invalid user data");  
	  
	    lua_pushnumber( L, (lua_Number)((*s)->GetAge()) );  
	  
	    return 1;  
	}  
	]]  

### main.cpp
	// MyCma.cpp : 定义控制台应用程序的入口点。  
	//  
	  
	//#include <SDKDDKVer.h>  
	#include <stdio.h>  
	#include <tchar.h>  
	#include <windows.h>  
	#include <string>  
	#include "lua.hpp"  
	#include "Student.h"  
	  
	  
	int _tmain(int argc, _TCHAR* argv[])  
	{  
	  
	    lua_State* L = luaL_newstate();  
	    luaL_openlibs(L);  
	    luaopen_student(L);  
	    luaL_dofile(L, "main.lua");  
	    lua_close(L);  
	  
	    system("pause");  
	  
	    return 0;  
	}  