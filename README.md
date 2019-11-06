# TheQKit

[![Version](https://img.shields.io/cocoapods/v/TheQKit.svg?style=flat)](https://cocoapods.org/pods/TheQKit)
[![Platform](https://img.shields.io/cocoapods/p/TheQKit.svg?style=flat)](https://cocoapods.org/pods/TheQKit)

# Example

To run the example project, clone the repo, and run `pod install` from the Example directory first.

## Login / Authentication and Setup instructions

1) Always change the bundle identifier to one that is accosiated with your developer account.

2) To use the Example app either Firebase or AccountKit (with "Sign in with Apple" coming soon) will need to be utilized and setup on the app and that login flow implemented. Once a user has been authenticated through these services can you pass the relative information to the SDK to create a user on The Q platform. 

    2a) For Firebase this includes adding the GoogleServices.plist file and any info.plist modifications
    2b) For AccountKit modifications to the app's info.plist are necessary
    
3) Sharing to snapchat is provided through the SDK, if the `SCSDKClientID` is found in the target's info.plist file it will use the account associated with that (or fail without it)


# Installation 

TheQKit is available through [CocoaPods](https://cocoapods.org). To install
it, simply add the following line to your Podfile:

```ruby
pod 'TheQKit'
```
or if adding from a local source

```ruby
pod 'TheQKit', :path => '<path to directory with the .podspec>'
```

Additionally in the podfile a few additions should be added to remove bitcode and enforce the most current architecture (both of these requirments are hopefully only temporary to this early version of the SDK)

```ruby
pre_install do |installer|
    Pod::Installer::Xcode::TargetValidator.send(:define_method, :verify_no_static_framework_transitive_dependencies) {}
end

post_install do |installer|
    installer.pods_project.targets.each do |target|
        target.build_configurations.each do |config|
            config.build_settings['ENABLE_BITCODE'] = 'NO'
            config.build_settings['ARCHS'] = '$(ARCHS_STANDARD_64_BIT)'
        end
    end
end
```

# Usage

## Initialize

Import TheQKit into AppDelegate.swift, and initialize TheQKit within `application:didFinishLaunchingWithOptions:` using the token provided by our support team

The `THEQKIT_TOKEN` allows for only this tenants games to be visable by TheQKit and is required.

```swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject: AnyObject]?) -> Bool {
    TheQKit.initialize(token: "<Provided Account Token>", baseUrl: "<Provide Base Url>")
}
```

## User Management

Integration with Account Kit must be done app side, after verification with those providers, pass along the token string, and account ID to login existing user  / enter user creation flow

Sign in with Apple
```swift
if let token = appleIDCredential.identityToken?.base64EncodedString() {
    let userIdentifier = appleIDCredential.user
    
    TheQKit.LoginQUserWithApple(userID: userIdentifier, identityString: token) { (success) in
        //success : Bool ... if user successfully is logged in, if no user exist will be false and account creation flow launches
    }
}
```

Account Kit: *DEPRECATED: with AccountKit shuting down, this method will go away, though avaliable for existing users or apps that are not setup for firebase phone #*
```swift
let token = AKFAccountKit(responseType: .accessToken).currentAccessToken
let accountID:String = token!.accountID
let tokenString:String = token!.tokenString

TheQKit.LoginQUserWithAK(accountID: accountID, tokenString: tokenString) { (success) in
    //success : Bool ... if user successfully is logged in, if no user exist will be false and account creation flow launches
}
```

Firebase: has a few different ways to accomplish the same thing, all ultimately end up with a user ID and token that get passed on to the TheQKit
```swift
extension YourVC: FUIAuthDelegate {
    func authUI(_ authUI: FUIAuth, didSignInWith authDataResult: AuthDataResult?, url: URL?, error: Error?) {

        if let user = authDataResult?.user {
            user.getIDTokenResult(completion: { (result, error) in
                let token = result?.token
                let userId:String = user.uid

                TheQKit.LoginQUserWithFirebase(userId: userId, tokenString: token) { (success) in
                    //success : Bool ... if user successfully is logged in, if no user exist will be false and account creation flow launches
                }
            })
        }
    }
}
```

Additionally you can skip the username selection and provide one the user may already have associated with your app. If already taken a number will incrementally be added

```swift    
TheQKit.LoginQUserWithApple(userID: userIdentifier, identityString: token, username : "username") { (success) in
     //success : Bool ... if user successfully is logged in, no account creation flow
}

*DEPRACTED*
TheQKit.LoginQUserWithAK(accountID: "accountID", tokenString: "tokenString", username : "username") { (success) in
    //success : Bool ... if user successfully is logged in, no account creation flow
}

TheQKit.LoginQUserWithFirebase(userId: "uid", tokenString: "tokenString", username : "username") { (success) in
    //success : Bool ... if user successfully is logged in, if no user exist will be false and account creation flow launches
}
```

Logout and clear stored user

```swift
TheQKit.LogoutQUser()
```

## Play Games - With UI
A schedule controller that will display game "cards" can be populated into any container view. These cards can be clicked on when the game is active and a logged in user exist.
```swift
override func prepare(for segue: UIStoryboardSegue, sender: Any?) {
    if segue.identifier == "name_of_your_seque" {
        let connectContainerViewController = segue.destination as UIViewController
        TheQKit.showCardsController(fromViewController: connectContainerViewController)
    }
}
```

## Play Games - No UI
To play a game with The Q Kit, after initilizing you will have to check and launch games with. 
```swift
TheQKit.CheckForGames { (isActive, gamesArray) in
    //isActive : Bool ... are any games returned currently active
    //gamesArray : [TQKGame]? ... active and non active games
}
TheQKit.LaunchGame(theGame : TQKGame) //Checks if specified game is active and launches it
TheQKit.LaunchActiveGame()            //checks for an active game and launches it if avaliable
```

## Customize Color Theme
Games have a default color theme, if you wish to over ride that, pass in the colorCode as a hex string to the launch functions
```swift
TheQKit.LaunchGame(theGame : TQKGame, colorCode : "#E63462") 
TheQKit.LaunchActiveGame(colorCode : "#E63462")         
```

## Get $ 

Users who win money, can request to cashout. To begin this request call the method below. A dialogue asking for the paypal email will show, then success/failure will show after the request

```swift
TheQKit.CashOutWithUI()
```

Or pass an email to this method to bypass the default dialogue

```swift
TheQKit.CashOutNoUI(email: String)
```

## Observable Keys 

Events that happen during the game; add an observer to add custom event tracking. Add the extension in for easy use or use the string literal. Each event return a [String:Any] dictionry of metadata relevant to the event


```swift 
NotificationCenter.default.addObserver(self, selector: #selector(errorSubmittingAnswer), name: .errorSubmittingAnswer, object: nil)

extension Notification.Name {
    static let choiceSelected = Notification.Name("TQK_GAME_CHOICE_SELECTED")
    static let errorSubmittingAnswer = Notification.Name("TQK_GAME_ERROR_SUB_ANSWER")
    static let screenRecordingDetected = Notification.Name("TQK_GAME_SCREEN_RECORDING")
    static let airplayDetected = Notification.Name("TQK_GAME_AIRPLAY")
    static let correctAnswerSubmitted = Notification.Name("TQK_GAME_CORRECT_ANS_SUB")
    static let incorrectAnswerSubmitted = Notification.Name("TQK_GAME_WRONG_ANS_SUB")
    static let enteredGame = Notification.Name("TQK_ENTERED_GAME")
    static let gameWon = Notification.Name("TQK_GAME_WON")
}
```

## Known Issues, Dependency Awareness, and Goals

This is first release of our SDK for playing The Q network games; issues are expected and currently the dependency list is larger than it should be. In future versions we would like to trim down the number of dependencies this framework requires.

## Author

Spohn, spohn@stre.am and the lovely people at Stream Live, Inc.

## License

Copyright (c) 2019 Stream Live, Inc.
