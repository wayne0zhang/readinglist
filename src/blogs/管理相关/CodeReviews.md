> ref: [Code review Best Practices](https://medium.com/palantir/code-review-best-practices-19e02780015f)

文章将了以下内容：

- 3w：why、what、when 进行 code review
- code review 之前的准备
- 执行 code review
- CRs 示例

### 1.为什么进行CR
- 代码编写者会规范自己的代码，也会觉得复查别人的代码有一种自豪感；
- 有益于分享知识
	- 项目中的其他人知道全部的功能
	- 写代码的人使用的算法或技术会被全员学习
	- 加强团队内沟通
- 保证代码风格统一，减少bug
- 简洁的代码不止易于复查，也能减少bug

### 2.复查什么
这个需要在时间和质量之间做选择。有的团队可能仅关注合并和主分支的功能，其他团队可能会对所有代码进行审查。

但是无论范围是什么，需要确定的是，所有人的代码都需要被复查。即使是毫无瑕疵的代码，在审查的过程中，其他人会学到很多编码的好习惯。

### 3.上线时再进行吗
一般来说复查是在自测、集成测试之后，需要把代码合到主分支之前进行。

### 4.复查之前的准备
- 提交者声明代码范围、内容的多少
- 完整的、自测通过之后的代码
- 代码重构不应该影响原有功能，相反，增加功能时也不应该重构其他代码。
	- 重构的代码会不重视
	- 大量的重构会不好做`遴选`,`刷新提交`等 git 功能

### 5.提交声明（commit message）
- 总结
- 描述做了什么以及怎么做的

### 6.复查人
- 1-2个熟悉项目的人，其中一个应该资深
- 不应该引入其他问题，像代码风格之类的讨论

### 7.code review
- 代码功能
	- 提交的代码实现了功能吗
	- 问问题
- 实现
	- 你会如何实现
	- 可以抽象吗
	- 像对手一样去复查，但是态度要友好
	- 考虑已实现库，减少重复造轮子
	- 实现是否改动了依赖，为什么
- 易读性
	- 你花多长时间读懂了？
	- 是否符合团队的编码风格
	- 是否有未完成的todo
- 可维护性
	- 看测试代码
	- 是否向后兼容
	- 是否需要集成测试
	- 提交反馈：是否有逻辑问题？
	- 文档是否更新了
- 安全性
	- 符合安全规范
- 注释：简洁、友好

### 8.反馈
- 要有反馈，即使是一句`done`
- 如果有修改，需要再次`review`

### 9.复查示例
#### 9.1 命名不一致
```
class MyClass {
  private int countTotalPageVisits; //R: name variables consistently
  private int uniqueUsersCount;
}
```

#### 9.2 方法声明不一致
```
interface MyInterface {
  /** Returns {@link Optional#empty} if s cannot be extracted. */
  public Optional<String> extractString(String s);
  /** Returns null if {@code s} cannot be rewritten. */
  //R: should harmonize return values: use Optional<> here, too
  public String rewriteString(String s);
}
```
#### 9.3 重复造轮子
```
//R: remove and replace by Guava's MapJoiner
String joinAndConcatenate(Map<String, String> map, String keyValueSeparator, String keySeparator);
```
#### 9.4 bugs
```
//R: This performs numIterations+1 iterations, is that intentional?
//   If it is, consider changing the numIterations semantics?
for (int i = 0; i <= numIterations; ++i) {
  ...
}
```
#### 9.5 架构考虑
```
otherService.call(); //R: I think we should avoid the dependency on OtherService. Can we discuss this in person?
```
	
	
	
	
	
	
	
	
	
	
	