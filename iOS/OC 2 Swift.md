### OC继承 VS Swift协议
在OC中协议是不可扩展的，所以在某些情况下不得不使用继承。以创建的ViewController为例为了让他们都有某些特性，我需要这样大致要写一个超类，然后创建的ViewController都继承自它。这种方法行之有效，但是类一单继承即拥有父类所有特性，

```objc
//OC
@interface CMBaseViewController ()
@property (nonatomic, strong) UITapGestureRecognizer *tapGesture;

@end

@implementation CMBaseViewController
- (void)viewDidLoad
{
    [super viewDidLoad];
    [self bindViewModel];
    [self bindView];
    [self layoutPageSubviews];
    
}

- (void)navigationBarClean:(BOOL)clean
{
    [self.navigationController.navigationBar setBackgroundImage:clean ? [UIImage new] : nil forBarMetrics:UIBarMetricsDefault];
    self.navigationController.navigationBar.shadowImage = clean ? [UIImage new] : nil;
}

- (void)addLeftButtonItem
{
    UIBarButtonItem *backButtonItem = [[UIBarButtonItem alloc] initWithImage:[UIImage imageNamed:@"icn_nav_back_nor"] style:UIBarButtonItemStylePlain target:self action:@selector(backBarButtonAction)];
    backButtonItem.tintColor = [UIColor whiteColor];
    UIBarButtonItem *leftSpaceItem = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemFixedSpace target:nil action:nil];
    leftSpaceItem.width = -10.0f;
    self.navigationItem.leftBarButtonItems = @[leftSpaceItem,backButtonItem];
}

- (void)addRightButtonItem {}
- (void)bindViewModel{}
- (void)bindView{}
- (void)layoutPageSubviews{}
- (void)backBarButtonAction
{
    [self.navigationController popViewControllerAnimated:YES];
}


- (RACSignal *)addKeyBoardWillShowSignal
{
    return [[RACSignal merge:@[
                               [[NSNotificationCenter defaultCenter] rac_addObserverForName:UIKeyboardWillShowNotification object:nil],
                               [[NSNotificationCenter defaultCenter] rac_addObserverForName:UIKeyboardWillHideNotification object:nil]
                               ]] map:^id(NSNotification* notification) {
        NSDictionary *userInfo = notification.userInfo;
        double duration = [userInfo[UIKeyboardAnimationDurationUserInfoKey] floatValue];
        CGRect keyboardF = [userInfo[UIKeyboardFrameEndUserInfoKey] CGRectValue];
        return RACTuplePack(@(duration),[NSValue valueWithCGRect:keyboardF]);
    }];
}

#pragma mark - property
- (void)addTapGesture
{
    [self.view addGestureRecognizer:self.tapGesture];
}

- (UITapGestureRecognizer *)tapGesture
{
    if (!_tapGesture) {
        _tapGesture = [UITapGestureRecognizer new];
        _tapGestureSignal = _tapGesture.rac_gestureSignal;
        _tapGesture.delegate = self;
    }
    return _tapGesture;
}

@end
```

在Swift中沿袭了OC的Category,并且对Protocol也支持此功能，上面的基类就可以转换成这样

```swift
protocol BaseViewProtocol {
    
    func initView();
    
    func layoutPageSubviews()
    
    func bindViewModel()
    
    func bindView()
    
}

protocol BaseViewGesture {
    func addTapGesture(view: UIView) -> UITapGestureRecognizer
}


protocol BaseViewNavigation {

}

extension BaseViewProtocol {
    func start() {
        initView()
        layoutPageSubviews()
        bindView()
        bindViewModel()
    }
}

extension BaseViewProtocol where Self: BaseViewGesture {
    func addTapGesture(view: UIView) -> UITapGestureRecognizer {
        let tapGesture = UITapGestureRecognizer()
        view.addGestureRecognizer(tapGesture)
        return tapGesture
    }
}
```

BaseViewProtocol只负责基本的公共构造方法，如果想在ViewController中添加一个gesture,只需要ViewController同时遵从BaseViewProtocol和BaseViewGesture两个协议才会拥有此特性

### Swift enum
swift的enum要比oc灵活的多，能够添加方法、拓展、遵从协议。参考了一下Moya这个库，在OC中写的适配器类可以这样

```swift
protocol LineTypeProtocol {
    func lineTypeData() -> (String, String, String,UIKeyboardType)
    
    func lineTypeTextLength(range: NSRange) -> Bool
}

struct CMLoginLineModel {
    var left: String!
    var center: String!
    var right: String!
    var keyBoardType: UIKeyboardType!
    var lineType: LineType!
    
    init(lineType: LineType) {
        self.lineType = lineType
        let value = self.lineType.lineTypeData()
        self.left = value.0
        self.center = value.1
        self.right = value.2
        self.keyBoardType = value.3
        
    }
}

enum LineType {
    case A4
    case Boss
    case Password
    case Phone
    case CheckNumber
}

extension LineType: LineTypeProtocol {
    func lineTypeData() -> (String, String, String, UIKeyboardType) {
        switch self {
        case .A4:
            return ("icon_login_user_activation","4A账号","icon_login_arrowdown_pressed", UIKeyboardType.default)
        case .Boss:
            return ("icon_login_user_activation","BOSS工号","icon_login_arrowdown_pressed", UIKeyboardType.default)
        case .CheckNumber:
            return ("icon_login_code_activation","短信验证码","获取", UIKeyboardType.numberPad)
        case .Password:
            return ("icon_login_password_activation","密码","",UIKeyboardType.default)
        case .Phone:
            return ("icon_login_phone_activation","手机号","icon_login_arrowdown_pressed",UIKeyboardType.numberPad)
        }
    }
    
    func lineTypeTextLength(range: NSRange) -> Bool {
        switch self {
        case .Phone:
            return  !(range.location > 12)
        case .CheckNumber:
            return !(range.location > 5)
        default:
            return true
        }
    }
}
```

### Swift Timer
Swift的NSTimer已经不推荐，取而代之的是DispatchSource.makeTimerSource(_:)

```swift
var loadingTimer = DispatchSource.makeTimerSource(queue: DispatchQueue(label: "com.smrz.loadingTimer"))
loadingTimer.schedule(deadline: .now(), repeating: .milliseconds(800))
loadingTimer.setEventHandler(handler: {
  if CMCheckNumberButton.str.count >= 7 {
     CMCheckNumberButton.str = "正在获取"
  }else {
     CMCheckNumberButton.str.append(".")
  }
  DispatchQueue.main.async(execute: {
    self.changeButtonEnable(enable: false)
    self.setButtonShow(value: CMCheckNumberButton.str)
    self.contentHorizontalAlignment = .left
  })
})
loadingTimer.cancel()
```

### ReactiveCocoa 6的属性监听
这个比较坑github上的实例是这样的

```swift
//这个#keyPath(person.name)看似是person实例的属性但其实。。。。这样莫名其妙编译时就会报错
let property = DynamicProperty<String>(object: person, keyPath: #keyPath(person.name))

//#keyPath正确的应该这样
let property = DynamicProperty<Bool>(object: loginButton, keyPath: #keyPath(UIButton.isEnabled))
```

**注意**
若在Swift的类中声明了一个变量，想使用KVO监听该变量，需要在声明变量的时添加dynamic关键字

### ReactiveCocoa Method Swizzing
ReactiveCocoa从正式迁移到Swift开始就已经代理方法迁移跟之前不太一，只能响应拦截方法的调用，而获取不到入参

```objc
//OC
_scrollViewSignal = [[self rac_signalForSelector:@selector(scrollViewDidEndDecelerating:)
                                            fromProtocol:@protocol(UIScrollViewDelegate)]
                             map:^id(RACTuple *value) {
                                 @strongify(self);
                                 UIScrollView *scrollView = (UIScrollView *)value.first;
                                 return RACTuplePack(self,
                                                     @((int)scrollView.contentOffset.x / (int)CGRectGetWidth(scrollView.frame))
                                                     );
                             }];
```

```swift
 //Swift
 scrollviewDidScroll =  self.reactive
            .trigger(for: #selector(scrollViewDidEndDragging(_:willDecelerate:)))
            .map { [unowned self] () -> (CMLoginScrollView,CGFloat) in
                return (self, self.contentOffset.x / self.frame.width)
        }
```

# 未完待续。。。。。。。





