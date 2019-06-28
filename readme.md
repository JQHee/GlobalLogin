iOS 全局做登录的逻辑

```
// 目标VC
var targetVC: UIViewController?
// 目标参数
var targetParams: [String: Any] = [String: Any]()
// 跳转类型
var jumpType = 0
// 当前VC
var currentVC: UIViewController?
```

```swift
deinit {
       NotificationCenter.default.removeObserver(self)
    }
    
    func viewBindEvents() {
        // 检测到未登录调用
         NotificationCenter.default.addObserver(self, selector: #selector(loginAction(noti:)), name: NSNotification.Name.init("Login"), object: nil)
        // 登录成功时调用
        NotificationCenter.default.addObserver(self, selector: #selector(loginSuccess(noti:)), name: NSNotification.Name.init("LoginSuccess"), object: nil)
    }
    // 全局处理登录
    @objc func loginAction(noti: Notification) {
        guard let topVC = self.topViewController() else {
            return
        }
        currentVC = topVC
        // print(noti.object ?? [:])
        let notiInfo = noti.object ?? [:]
        if let tempNotiInfo = notiInfo as? [String: Any] {
            targetParams = tempNotiInfo
            jumpType = (tempNotiInfo["jumpType"] as? Int) ?? 0
            targetVC = tempNotiInfo["targetVC"] as? UIViewController
        } else { // 置空
            targetParams = [:]
            jumpType = 1
            targetVC = nil
        }
       
        if let _ = topVC as? LoginViewController {
            debugPrint("LoginVC")
            return
        }
        let loginNav = BaseNavigationController(rootViewController: LoginViewController())
        self.presentVC(loginNav)

    }
    
    @objc func loginSuccess(noti: Notification) {
        // 要刷新页面
        guard let tempCurrentVC = currentVC else {
            return
        }
        
        if jumpType == 1 { // push跳转
            guard let tempTargetVC = targetVC else {
                return
            }
            tempCurrentVC.navigationController?.pushViewController(tempTargetVC, animated: true)
            
        } else if jumpType == 2 { // modal 跳转
            
        } else { // 进入目标VC才跳转（登录过期）直接刷新页面
            if tempCurrentVC.isKind(of: HFCollectionListMainViewController.self) {
                (tempCurrentVC as? HFCollectionListMainViewController )?.refreshSubViewDatas(param: targetParams)
            }
        }
        
    }
```

```swift
    // 获取当前的VC
    func topViewController(_ viewController: UIViewController? = nil) -> UIViewController? {
        let viewController = viewController ?? UIApplication.shared.keyWindow?.rootViewController
        
        if let navigationController = viewController as? UINavigationController,
            !navigationController.viewControllers.isEmpty {
            return self.topViewController(navigationController.viewControllers.last)
            
        } else if let tabBarController = viewController as? UITabBarController,
            let selectedController = tabBarController.selectedViewController {
            return self.topViewController(selectedController)
            
        } else if let presentedController = viewController?.presentedViewController {
            return self.topViewController(presentedController)
            
        }
        
        return viewController
    }
```

需要登录才跳转的页面
如：登录成功后才跳转到收藏页面
```swift
        // 我的收藏
        let VC = HFCollectionListMainViewController.loadStoryboard(.collection)
        if MLUserModel.shared.isLogin() {
            navigationController?.pushViewController(VC, animated: true)
        } else {
            // 0为normal  1为push  2为model
            NotificationCenter.default.post(name: NSNotification.Name.init("Login"), object: ["targetVC": VC, "jumpType": 1])
        }
```

登录成功

```
 NotificationCenter.default.post(name: NSNotification.Name.init("LoginSuccess"), object: nil)
```

接口登录全局状态码拦截
未登录则弹窗登录框
```swift
    static func interceptStatusCode(json: JSON) {
        if let _ = json["code"].int {
            let code = json["code"].intValue
            if code == 101 || code == 100 { // 登录过期 (100 未登录 101登录过期)
                MLUserModel.shared.clear()
                NotificationCenter.default.post(name: NSNotification.Name.init("Login"), object: nil)
            }
        }
    }
```


