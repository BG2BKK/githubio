+++
date = "2016-05-11T11:36:47+08:00"
draft = true
title = "leetcode解题过程中的思考"

+++


* 8. String to Integer (atoi)
	* https://leetcode.com/problems/string-to-integer-atoi/
	* 何海涛代码

```cpp
// StringToInt.cpp : Defines the entry point for the console application.
//
// 《剑指Offer——名企面试官精讲典型编程题》代码
// 著作权所有者：何海涛
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <iostream>
using namespace std;

long long StrToIntCore(const char* str, bool minus);
enum Status {kValid = 0, kInvalid};
int g_nStatus = kValid;
int status = kValid;
int StrToInt(const char* str)
{
    g_nStatus = kInvalid;
    long long num = 0;
    if(str != NULL && *str != '\0') 
    {
        bool minus = false;
        if(*str == '+')
            str ++;
        else if(*str == '-') 
        {
            str ++;
            minus = true;
        }
        if(*str != '\0') 
        {
            num = StrToIntCore(str, minus);
        }
    }
    return (int)num;
}

// 要求atoi，integer为32位整数，范围是  -2147483648(0x80000000) ~ 2147483647(0x7fffffff)
// 由于要判断是否溢出，所以采用long long类型来实现加减，范围是 -9223372036854775808 ~ 9223372036854775807
// 最后返回值时，采用(int)num来将long long强制转换为int，由于num范围就是int类型这么大，所以不会出错

// 判断是否溢出时 num > 0x7fffffff 可以直接判断，毕竟num可以很大

// 判断输入str的有效性

long long StrToIntCore(const char* digit, bool minus)
{
    long long num = 0;
    while(*digit != '\0') 
    {
        if(*digit >= '0' && *digit <= '9') 
        {
            int flag = minus ? -1 : 1;
            num = num * 10 + flag * (*digit - '0');
            if((!minus && num > 0x7FFFFFFF) 
                || (minus && num < (signed int)0x80000000))
            {
                num = 0;
                break;
            }
            digit++;
        }
        else 
        {
            num = 0;
            break;
        }
    }
    if(*digit == '\0') 
    {
        g_nStatus = kValid;
    }
    return num;
}

// ====================测试代码====================

void Test(char* string)
{
    int result = StrToInt(string);
    if(result == 0 && g_nStatus == kInvalid)
        printf("the input %s is invalid.\n", string);
    else
        printf("number for %s is: %d.\n", string, result);
}

int main(int argc, char * argv[])
{
//    Test(NULL);
    Test("");
    Test("123");
    Test("+123");
    
    Test("-123");
    Test("1a33");
    Test("+0");
    Test("-0");
    //有效的最大正整数, 0x7FFFFFFF
    Test("+2147483647");    
    Test("-2147483647");
    Test("+2147483648");
    //有效的最小负整数, 0x80000000
    Test("-2147483648");    
    Test("+2147483649");
    Test("-2147483649");
    Test("+");
    Test("-");
    return 0;
}
```

	* 我的狗尾续貂之作
```cpp
long long str2int(string str){
	
	const char *s = str.c_str();
	if(!s || *s == '\0'){
		return 0;
	}

	bool minus = false;
	long long num = 0;
	if(*s == '+'){
		minus = false;
		s++;
	} else if ( *s == '-') {
		minus = true;
		s++;
	}

	while(*s == '0')
		s++;
	while(*s != '\0'){
		if( *s >= '0' && *s <= '9'){
			int digit = *s - '0';
			int flag = minus ? -1 : 1;
			num = num * 10 + flag * digit;

			if( (!minus &&num > 0x7fffffff) 
					|| (minus && num < (signed int)0x80000000)){
				num = 0;
				status = kInvalid;
				break;
			} else {
				s++;
			}
		} else {
			status = kInvalid;
			num = 0;
			break;
		}
	}

	if(*s == '\0') {
		status = kValid;
	}
	return num;
}
void Test(string str)
{
    long long result = str2int(str);
    if(result == 0 && status == kInvalid)
        printf("the input %s is invalid.\n", str.c_str());
    else
        printf("number for %s is: %d.\n", str.c_str(), result);
}
```

	* leetcode accept 代码

```cpp

class Solution {
	public:
		int myAtoi(string str) {
			const char *s = str.c_str();
			if(!s || *s == '\0'){
				return 0;
			}
			while(*s == ' ') s++;
			bool minus = false;
			long long num = 0;
			if(*s == '+'){
				minus = false;
				s++;
			} else if ( *s == '-') {
				minus = true;
				s++;
			}
			while(*s != '\0'){
				if( *s >= '0' && *s <= '9'){
					if(*s == '0' && num == 0) { s++; continue;}
					int digit = *s - '0';
					int flag = minus ? -1 : 1;
					num = num * 10 + flag * digit;
					if (!minus &&num > 0x7fffffff) {
						return INT_MAX;
					} else if (minus && num < (signed int)0x80000000){
						return INT_MIN;
					}
					s++;
				} else {
					break;
				}
			}
			return (int)num;
		}
};

```
