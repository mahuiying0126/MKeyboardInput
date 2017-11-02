# MKeyboardInput
简书地址:http://www.jianshu.com/p/df69553e20c0

下面就给大家分享一下我做这个界面的两个思路：
##### 思路一：

 将要弹出的视图做为一个 textFiled 的`inputAccessoryView`这样做的好处是不用处理弹出视图的动画与键盘动画同步，也不用监听键盘的弹出收回事件。

我在控制器里声明一个 textFiled 

`@property (nonatomic, strong) UITextField *textFile;`

在初始化的时候只` alloc`而不给` Frame`

	self.textFile = [UITextField new];
	//加载到 view 中让 view 持有避免回收
   	[self.view addSubview:self.textFile];
因为想着做一个类似微信回复评论的那种，可以输入多行有高度限制，就在自定义 view 里面用到了 textview 的代理，第一步想到的代理就是用`ViewDidChange`方法

	- (void)textViewDidChange:(UITextView *)textView；

通过这个代理来计算输入文字的宽度，计算文字宽度是用的系统的字体`Attribute`属性

	CGSize  textSize = [textView.text sizeWithAttributes:@{NSFontAttributeName:[UIFont systemFontOfSize:FontSize]}];

给 textview 一个固定宽度（或者在初始化视图时获取 textview 的宽度），通过两者比较来判断输入的文字是否换行，如果换行就把 textview 的高度和视图本身的高度改变，改变的大小就是文字换行的大小。

在控制器里，初始化自定义视图：

	self.inputAccessoryView = [[MReplyCommentView alloc] initWithFrame: CGRectMake(0, 0, windowRect.size.width, 50)];
    
    self.textField.inputAccessoryView = self.inputAccessoryView;

这样我在点评论按钮的时候将` textFiled`变成键盘第一响应就行了，键盘带着输入框就弹出来了，很流畅。等等，不对啊，自定义视图里的 textview 并没有进入编辑状态，也就是没有光标闪烁。然后我在自定义视图`init`方法里加了键盘的通知`UIKeyboardWillShowNotification`：

	[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(keyboardWillShow) name:UIKeyboardWillShowNotification object:nil];

	- (void)keyboardWillShow {
	    if (![self.textView isFirstResponder]) {
	        [self.textView becomeFirstResponder];
	    }
	}

通过 if 判断来判断 textv 是否为键盘第一响应者。

**输入文字换行遇到的问题**

因为自定义视图的高度是写死的为50，textview 的高度也是写死的是30，(16号字体时 textview 高度为36)在换行的时候光标总是下移，删除字体换行的时候光标会上移，每当输入当前行最后一个字换到下一行时光标就跑的很离谱。字体宽度我是通过计算字体的宽度来计算的，高度我是这样计算的：

	// TextView 大小
	CGRect bounds = self.textView.bounds;
	CGSize maxSize = CGSizeMake(bounds.size.width, CGFLOAT_MAX);
	//用这个方法将得到的尺寸转化为 textview 最适合的尺寸
	CGSize newSize = [self.textView sizeThatFits:maxSize];
	self.textView.frame = CGRectMake(10, 10, bounds.size.width, newSize.height);

但是每次一到换行的时候光标就移动的很离谱，而且 textview 有晃动，我以为是这个`sizeThatFits `方法转化的不是很准确，我就打断点一点一点看数据，发现也是正确的，这让我感到很困惑，我在网上查了下相关资料，都推荐让用kvo 去监听 textview 的`contentSize`属性，我也借来用了一下：

	- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context{
	    UITextView * view =(UITextView *)object;
	    if ([keyPath isEqualToString:@"contentSize"]) {
	        
	        CGFloat height = view.contentSize.height;
	        if (height > 112) {
	            height = 112;
	        }
	        CGRect textViewFrame = self.textView.frame;
	        textViewFrame.size.height = height;
	        self.textView.frame = textViewFrame;
	        [self updateSelfOfTextViewSize];
	    }
	}

发现比原来的要好很多，而且这个监听方法比 textview 的代理要好用的多，代理是没次输入字的时候都会去计算宽度，而这个监听只有 textview 换行的时候才会调用，而且换行高度也是字体高度。现在 textview 的光标没有跑那么离谱了，但是还是有 frame 设置的问题，我百思不得其解，我随意输入几行字，用 xcode 打开app 的图层，发现光标的高度在16号字体下竟然是36而不是30，导致每次换行都有误差，设置好高度换行问题都迎刃而解了。

**点击页面空白处不能回收键盘问题**

我在控制器中设置取消编辑事件发现键盘不能回收，

	- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
	    [self.view endEditing:YES];
	}

而且在点击控制器页面的时候发现方法都不执行，可能是我用一个三方` IQKeyboardManager`的原因，也可能是图层的原因，我也没有追究到底是哪里的原因，我放弃了这种做法，因为浪费的时间已经很多了。

**思路二：**

自定义视图重写`-(instancetype)init`方法，将textview的初始化放在这里，分别加入键盘出现监听：`UIKeyboardWillShowNotification`，和消失监听`UIKeyboardWillHideNotification`，通过对键盘的监听来设置自定视图的`Frame`和 textview 的动画，再加上上次踩的坑都一起设置完毕，run 了一下达到了预想的结果，可能是因为三方库` IQKeyboard`的原因，每次弹出视图，控制器视图会滚动，所以在`viewWillAppear`和`viewWillDisappear`中加入了`[IQKeyboardManager sharedManager].enable = NO;`将 IQ 的作用取消。具体代码可以看一下 demo [点我下载](https://github.com/mahuiying0126/MKeyboardInput)。

**总结**

这个功能也不是很复杂，花费了一天时间才搞定。主要用的知识点：对 textview 做`contentSize`kvo 监听，通过`contentSize.height`来设定高度要比自己计算准确的多，一定要根据字号高度把 textview 的初始化高度计算准确，要不然光标会跳跃；通过监听键盘来做动画改变 frame。

**最后再说一点**

此 demo 很简单，只适应小工程，有兴趣的同学可以试一下，如果你的工程里比如有界面套界面，列表套列表，层级关系比较多，想弹键盘视图可以将自定义 view 设置成单例，这样就好控制了。设置单例的时候要把`-(void)close`方法中的`[[NSNotificationCenter defaultCenter] removeObserver:self];`代码注释掉.
