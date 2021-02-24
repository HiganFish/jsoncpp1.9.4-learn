时隔1年再次回来阅读jsoncpp代码, 看看现在有什么收获. 

昨天晚上发现了 https://github.com/Tencent/rapidjson 这就去看看 溜溜溜

[2020年3月的笔记](https://github.com/HiganFish/JsoncppLearn)

jsoncpp的两大基础功能

- json的生成

- json的序列化
- 字符串到json的反序列化



jsoncpp中`json`的`key`和`value`都是作为`value对象`来看待

一个value对象中含有多个kv键值对

本文中默认的k和v为json概念中的key和value

而Value和value则默认为jsoncpp中抽象出的value对象

# json的生成

从一段代码开始

```c++
int main()
{
   Json::Value root;
   Json::Value data;
   root["action"] = "run"; // run 1 data 最终都是是或者转换成了Value对象
   data["number"] = 1;
   root["data"] = data;

   Json::StreamWriterBuilder builder;
   builder["indentation"] = ""; // 这里加了一行 去除换行和缩进
   const std::string json_file = Json::writeString(builder, root);
   std::cout << json_file << std::endl;

   return EXIT_SUCCESS;
}
```

`root["action"] = "run"`首先调用重载的`operator=` 可以在这里通过重载不同的类型 实现不同类型的key

```c++
Value& Value::operator[](const char* key)
{
	return resolveReference(key, key + strlen(key));
}
```

进入resolveReference查看

```c++
Value& Value::resolveReference(char const* key, char const* end)
{
	JSON_ASSERT_MESSAGE(
			type() == nullValue || type() == objectValue,
			"in Json::Value::resolveReference(key, end): requires objectValue");

	// 默认是nullValue 默认创建一个objectValue
	if (type() == nullValue)
		*this = Value(objectValue);

	// 包装字符串
	CZString actualKey(key, static_cast<unsigned>(end - key),
			CZString::duplicateOnCopy);
	// TODO 为什么使用了lower_bound而不是find 使用find还不用考虑 (*it).first == actualKey
    // 每个value对象中都维护者自己的map 极大地方便了json的嵌套 std::map<CZString, Value>
	auto it = value_.map_->lower_bound(actualKey);
	if (it != value_.map_->end() && (*it).first == actualKey)
		return (*it).second;

	ObjectValues::value_type defaultValue(actualKey, nullSingleton());
	it = value_.map_->insert(it, defaultValue);
	Value& value = (*it).second;
	return value;
}
```

**为什么使用了`lower_bound`?  明明使用find还能避免掉if的第二个条件. **

再往后看就能发现`insert`语句中再次使用了`it` 由于`lower_bound`返回的迭代器能够指导`insert`更加高效的插入的输入 所以这样写 而对于`find` 就算使用`insert`也会由于是`end`无法起到指导作用将会花费额外的`O(log N)`插入



最终这个函数找到了相同k 的`Value`或者创建了新的`Value` 然后返回



然后是Value的`operator=`重载

```c++
Value& Value::operator=(Value&& other)
{
	other.swap(*this);
	return *this;
}
```

可以看到这里使用了移动赋值运算符 交换了临时的other和this中的内部数据 减少了资源的创建和删除导致的浪费



而对于k `"run"`到`Value`的转换

```c++
Value::Value(const char* value)
{
	initBasic(stringValue, true); // 通过initBasic设置value的类型 并进行初始化
	JSON_ASSERT_MESSAGE(value != nullptr,
			"Null Value Passed to Value Constructor");
	value_.string_ = duplicateAndPrefixStringValue( // 保存
			value, static_cast<unsigned>(strlen(value)));
}

union ValueHolder // union的妙用 当然少不了搭配的value_type_
{
    LargestInt int_;
    LargestUInt uint_;
    double real_;
    bool bool_;
    char* string_; // if allocated_, ptr to { unsigned, char[] }.
    ObjectValues* map_;
} value_;

struct
{
    // Really a ValueType, but types should agree for bitfield packing.
    unsigned int value_type_: 8;
    // Unless allocated_, string_ must be null-terminated.
    unsigned int allocated_: 1;
} bits_;
```



json对象的生成就这样结束了 上面分析的是json的k v



# json对象到string的转换

有了json对象当然要进行json对象的序列化 保存到string中



首先来到了writeString函数中

```c++
String writeString(StreamWriter::Factory const& factory, Value const& root)
{
	OStringStream sout; // 一个std::ostringstream
	StreamWriterPtr const writer(factory.newStreamWriter()); // char* const a  和const char* a 前者a不能变 后者a的指向内容不能变
	writer->write(root, &sout);
	return sout.str();
}
```

这里使用了工厂方法通过之前读取的配置进行`writer`初始化 并设置初始值



然后进入重载后的`write`函数

```c++
int BuiltStyledStreamWriter::write(Value const& root, OStream* sout)
{
	sout_ = sout;
	addChildValues_ = false;
	indented_ = true;
	indentString_.clear();
	writeCommentBeforeValue(root);
	if (!indented_)
		writeIndent();
	indented_ = true;
	writeValue(root); // 主要是这里 前面和后面的暂时忽略 Indent意为缩进
	writeCommentAfterValueOnSameLine(root);
	*sout_ << endingLineFeedSymbol_;
	sout_ = nullptr;
	return 0;
}
```



进入`writeValue`函数一个超长的函数

```c++
void BuiltStyledStreamWriter::writeValue(Value const& value)
{
	switch (value.type())
	{
	case nullValue:
		pushValue(nullSymbol_);
		break;
	case intValue:
		pushValue(valueToString(value.asLargestInt()));
		break;
	case uintValue:
		pushValue(valueToString(value.asLargestUInt()));
		break;
	case realValue:
		pushValue(valueToString(value.asDouble(), useSpecialFloats_, precision_,
				precisionType_));
		break;
	case stringValue: // #2 第二次是这里 为了写入 "action" 对应的"run"
	{
		// Is NULL is possible for value.string_? No.
		char const* str;
		char const* end;
		bool ok = value.getString(&str, &end);
		if (ok)
			pushValue(valueToQuotedStringN(str, static_cast<unsigned>(end - str),
					emitUTF8_));
		else
			pushValue("");
		break;
	}
	case booleanValue:
		pushValue(valueToString(value.asBool()));
		break;
	case arrayValue:
		writeArrayValue(value);
		break;
	case objectValue: // #1 本例中首先会进入这里
	{
		Value::Members members(value.getMemberNames());
		if (members.empty())
			pushValue("{}");
		else // 走这里
		{
			writeWithIndent("{"); // objectValue 起手写入{
			indent();
			auto it = members.begin();
			for (;;)
			{
				String const& name = *it; // 取出Value对象中第一个kv的k的值 第一次为"action"
				Value const& childValue = value[name]; // 取出kv的v
				writeCommentBeforeValue(childValue);
				writeWithIndent(valueToQuotedStringN( // 写入带引号的"action"
						name.data(), static_cast<unsigned>(name.length()), emitUTF8_));
				*sout_ << colonSymbol_; // 分隔符 默认是 :
				writeValue(childValue); // 由于 Jsoncpp中kv中的kv都是抽象为了Value对象 这里进行递归写入
				if (++it == members.end()) 
				{
					writeCommentAfterValueOnSameLine(childValue);
					break; // 如果本Value对象中的KV已经写完则退出循环
				}
				*sout_ << ","; // 写完一个kv之后写入一个 ,
				writeCommentAfterValueOnSameLine(childValue);
			}
			unindent();
			writeWithIndent("}"); // Value对象KV写完后再 写入}
		}
	}
		break;
	}
}
```



# string到Json对象的转换



首先还是上官方的例子, 删去了一小部分代码

```c++
const std::string rawJson = R"({"Age": 20, "Name": "colin"})";
const auto rawJsonLength = static_cast<int>(rawJson.length());

JSONCPP_STRING err;
Json::Value root;

Json::CharReaderBuilder builder; // 工厂模式 用于加载配置类初始化CharReader
const std::unique_ptr<Json::CharReader> reader(builder.newCharReader());

// 通过 err来返回错误
if (!reader->parse(rawJson.c_str(), rawJson.c_str() + rawJsonLength, &root,
                   &err))
{
    std::cout << "error" << std::endl;
    return EXIT_FAILURE;
}
```

进入 parse函数

```c++
bool parse(char const* beginDoc, char const* endDoc, Value* root,
			String* errs) override
{
    bool ok = reader_.parse(beginDoc, endDoc, *root, collectComments_);
    if (errs)
    {
        *errs = reader_.getFormattedErrorMessages();
    }
    return ok;
}
```

这个函数很有意思, 按照我开始的想法 errs应该是传入到了底层或者保存了指针, 然而 他是在解析完之后 读取错误... 

继续进入parse函数

```c++
bool OurReader::parse(const char* beginDoc, const char* endDoc, Value& root,
		bool collectComments)
{
	if (!features_.allowComments_)
	{
		collectComments = false;
	}

	begin_ = beginDoc; // 将字符串保存
	end_ = endDoc;
	collectComments_ = collectComments;
	current_ = begin_;
	lastValueEnd_ = nullptr;
	lastValue_ = nullptr;
	commentsBefore_.clear();
	errors_.clear();
	while (!nodes_.empty())
		nodes_.pop();
	nodes_.push(&root); // 直接来看这里 第一次出现了root nodes->std::stack<Value*>

	skipBom(features_.skipBom_);
	bool successful = readValue(); // 在这里进行解析
	nodes_.pop();
    ...
	return successful;
}
```

进入readValue函数

```c++
bool OurReader::readValue()
{
	//  To preserve the old behaviour we cast size_t to int.
	if (nodes_.size() > features_.stackLimit_)
		throwRuntimeError("Exceeded stackLimit in readValue().");
	Token token;
	skipCommentTokens(token); // 这里读取token token是通过读取下一个保存的字符 来判断接下来的数据类型 { 代表 tokenObjectBegin } 代表结束等等
	bool successful = true;

	if (collectComments_ && !commentsBefore_.empty())
	{
		currentValue().setComment(commentsBefore_, commentBefore);
		commentsBefore_.clear();
	}

	switch (token.type_)
	{
	case tokenObjectBegin: // 第一次走这里
		successful = readObject(token); 
		currentValue().setOffsetLimit(current_ - begin_);
		break;
	case tokenNumber: // 然后这里读取了v  age对应的20
		successful = decodeNumber(token);
		break; 
	case tokenString: // name对应的 colin
		successful = decodeString(token);
		break;
	}
	return successful;
}
```

进入`readObject()`函数

```c++
bool OurReader::readObject(Token& token)
{
	Token tokenName;
	String name;
	Value init(objectValue);
	currentValue().swapPayload(init);
	currentValue().setOffsetStart(token.start_ - begin_);
	while (readToken(tokenName)) // 再次读取token 这里读取到的是 "arg" 代表字符串的token 字符串的token为双引号开始 然后双引号结束
	{
		bool initialTokenOk = true;
		if (tokenName.type_ == tokenString)
		{
			if (!decodeString(tokenName, name)) // 解析字符串 "age"转为了 age保存到了name之中 解析的时候调用了reserve扩充字符串cap
				return recoverFromError(tokenObjectEnd);
		}
		else if (tokenName.type_ == tokenNumber && features_.allowNumericKeys_)
		{
			Value numberName;
			if (!decodeNumber(tokenName, numberName))
				return recoverFromError(tokenObjectEnd);
			name = numberName.asString();
		}
		else
		{
			break;
		}

		if (features_.rejectDupKeys_ && currentValue().isMember(name)) // 读取到name后检测重复
		{
			String msg = "Duplicate key: '" + name + "'";
			return addErrorAndRecover(msg, tokenName, tokenObjectEnd);
		}

		Token colon;
		if (!readToken(colon) || colon.type_ != tokenMemberSeparator) // 去除掉 name后的 :
		{
			return addErrorAndRecover("Missing ':' after object member name", colon,
					tokenObjectEnd);
		}
		Value& value = currentValue()[name];
		nodes_.push(&value); // 接下来改解析这个name k对应的 v了 将其移动到栈顶
        bool ok = readValue(); // 继续调用readValue 本例中对应的数字 解析数字 则是读取 char 然后 - '0' + num * 10 这种方式
		nodes_.pop(); // 读取结束了value 出栈
		if (!ok) // error already set
			return recoverFromError(tokenObjectEnd);

		Token comma;
		if (!readToken(comma) || // 再次读取token 移除 ,
				(comma.type_ != tokenObjectEnd && comma.type_ != tokenArraySeparator &&
						comma.type_ != tokenComment))
		{
			return addErrorAndRecover("Missing ',' or '}' in object declaration",
					comma, tokenObjectEnd);
		}
		bool finalizeTokenOk = true;
		while (comma.type_ == tokenComment && finalizeTokenOk)
			finalizeTokenOk = readToken(comma);
		if (comma.type_ == tokenObjectEnd) // 这里会判断 是不是Object结束 不是则继续解析 如此反复 知道object解析结束
			return true;
	}
	return addErrorAndRecover("Missing '}' or object member name", tokenName,
			tokenObjectEnd);
}
```



`parse`进行初始化, 然后调用`readValue`进行json的解析, 根据读取到的token 来决定`readValue` 的走向

第一次需要读取到`{`标识Object的开始, 调用`readObject`

`readObject`则是先读取到`k` 判断下有无重复 然后移除掉`:` 再次解析`v`. 然后再次读取token 这里简单来说需要是`, (标识一个kv的结束)`或者是`} (标识Object的结束)` 通过栈来控制当前解析的Value;



预先压入了`root`然后读取到了`k`又压入了此`k`对应的Value  然后解析`v`结束后 出栈  栈中现在包含的就是预先压入的`root`了只不过这时候已经保存了一个`kv`了