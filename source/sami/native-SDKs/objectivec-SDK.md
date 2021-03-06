---
title: "Objective-C/iOS SDK"

---

#Objective-C/iOS SDK (beta)

This SDK helps you connect your iOS apps to SAMI. It exposes a number of methods to easily execute REST API calls to SAMI.

To help you learn the SDK, we will talk about our iOS demo app that uses the SDK. We will walk through the steps to setup the iOS demo app and explain the code that calls SDK APIs to interact with SAMI.

## Prerequisites

- [Xcode v6.1](https://developer.apple.com/xcode/) or above
- AFNetworking
- [CocoaPods](http://guides.cocoapods.org/using/getting-started.html)

The SDK uses AFNetworking library. The demo app uses CocoaPods to manage the dependency of the SDK on this library.

The SDK supports iOS version 6 and higher. The sample application was written for and tested in iOS 8. These instructions are for a Mac that is running Xcode v6.1 and above.

### Source code

Get the source code of the [Objective-C/iOS SDK](https://github.com/samsungsamiio/sami-ios) and the [iOS demo app](https://github.com/samsungsamiio/sami-ios-demo) from GitHub.

## Setup and installation

1. Create an Application in devportal.samsungsami.io:
  * The Redirect URI is set to 'ios-app://redirect'.
  * Choose "Client credentials, auth code, implicit" for OAuth 2.0 flow.
  * Under "PERMISSIONS", check "Read" for "Profile". 
  * Click the "Add Device Type" button. Choose "SAMI Demo IPhonePedometer" as the device type. Check "Read" and "Write" permissions for this device type.
  * Click the "Add Device Type" button again. Choose a few more device types that you want to play with using the demo app. Check "Read" permissions for these device types.
2. Install CocoaPods. See [this page](http://guides.cocoapods.org/using/getting-started.html) for instructions. From a terminal window, locate the SAMIClient directory of the demo app, and run `pod install`. This installs all the prerequisites like AFNetworking.
3. Double click `SAMIClient.xcworkspace` (not `SAMIClient.xcodeproj`) to open the Xcode project.
4. Import the SAMI SDK into the Xcode project of the demo app. In the Xcode project, drag the `client` folder of SAMI iOS SDK from the Finder window into `SAMIClient` group in Xcode.
![iOS SAMI Demo setup](/images/docs/sami/native-SDKs/ios-demo-add-sdk-to-prj.png){:.lightbox}
5. Copy the Application Client ID obtained in Step 1 into `SamiConstants.h` to replace `YOUR CLIENT APP ID`
![iOS SAMI Demo setup](/images/docs/sami/native-SDKs/ios-demo-modify-appid.png){:.lightbox}
6. Build the Application in XCode.
7. Run the Application in iOS Simulator.

## OAuth2 flow

iOS apps require the use of Implicit Grant flows. This involves launching a `UIWebView` and submitting a `GET` request to the Authorization endpoint: `https://accounts.samsungsami.io/authorize` with the following parameters URL-encoded:

|Name |Value
|--- |---
|`client_id`{:.param} |[Get Application ID from admin.samsungsami.io]
|`response_type`{:.param} |token
|`redirect_uri`{:.param} |ios-app://redirect
|`client`{:.param} |mobile

After the request is submitted via the `WebView`, the iOS app needs to 'trap' the redirect in order to capture the accessToken.

**SamiOAuth2ViewController**

~~~
#import "SamiOAuth2ViewController.h"
#import "SamiUserSession.h"
  
// @interface SamiOAuth2ViewController : UIViewController<UIWebViewDelegate>
@interface SamiOAuth2ViewController ()
  
@property (weak, nonatomic) IBOutlet UITextField *addressField;
@property (weak, nonatomic) IBOutlet UIWebView *webView;
  
@end
  
@implementation SamiOAuth2ViewController
  
- (void)viewDidLoad
{
    [super viewDidLoad];
    self.webView.delegate = self;
     
    //6. Create the authenticate string that we will use in our request to foursquare
    // we have to provide our client id and the same redirect uri that we used in setting up our app
    // The redirect uri can be any scheme we want it to be... it's not actually going anywhere as we plan to
    // intercept it and get the access token off of it
    NSString *authenticateURLString = [NSString stringWithFormat:@"https://accounts.samihub.com/authorize?client=mobile&client_id=%@&response_type=token&redirect_uri=%@",
      SAMI_CLIENT_ID,
      SAMI_REDIRECT_URL];
    //7. Make the request and load it into the webview
    NSURLRequest *request = [NSURLRequest requestWithURL:[NSURL URLWithString:authenticateURLString]];
     
    self.addressField.text = authenticateURLString;
    [self.webView loadRequest:request];   
}
  
- (void)didReceiveMemoryWarning
{
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}
  
#pragma mark - Web view delegate
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType {
     
    if([request.URL.scheme isEqualToString:@"ios-app"]){
        // 8. get the url and check for the access token in the callback url
        NSString *URLString = [[request URL] absoluteString];
        if ([URLString rangeOfString:@"access_token="].location != NSNotFound) {
            // 9. Store the access token in the user defaults
            NSString *accessToken = [[URLString componentsSeparatedByString:@"="] lastObject];
            [SamiUserSession sharedInstance].accessToken = accessToken;
            // 10. dismiss the view controller
            //[self.navigationController popToRootViewControllerAnimated:YES];
            [self dismissViewControllerAnimated:YES completion:nil];
        }
    }
    return YES;
}
  
@end
~~~

After getting the accessToken, the app should store in securely and re-use it until its expiry.


## Get user info

The first task is to use the access token to obtain the user's information: ID, full name, email, etc.

This is accomplished using the `SamiUsersApi.h` class in the SAMI Objective-C library. The API calls are executed asynchronously, and the error or response needs to be retrieved in the completion block, as below:

**SamiViewController.m**

~~~
#import "SamiUsersApi.h"
  
  
@implementation SamiViewController
  
- (void) validateAccessToken: (NSString *) accessToken {
  SamiUsersApi *usersApi = [[SamiUsersApi alloc] init];
  [usersApi addHeader:[NSString stringWithFormat:@"Bearer %@", accessToken] forKey:@"Authorization"];
   
  [usersApi selfWithCompletionBlock:^(SamiUserEnvelope *output, NSError *error) {
    if (!error) {
      NSLog(@"%@", output.data);
    }
  }
}
  
@end
~~~

## Register device

After obtaining the user's ID, other user information like the user's devices, device types and properties can be retrieved. 

To 'register' the iPhone as a device, the iOS app needs to use the `SamiDevicesApi.h` addDevice call, passing the appropriate device type ID, below:

**SamiDevicesTableViewController.m**

~~~
- (IBAction)registerPhoneAsDevice:(id)sender {
    SamiDevicesApi * api = [[SamiDevicesApi alloc] init];
    [api addHeader:[NSString stringWithFormat:@"Bearer %@", self.accessToken] forKey:@"Authorization"];
  
    SamiDevice* device = [[SamiDevice alloc] init];
    device.name = @"<Phone UDID>";
    device.uid = @"<Sami User ID>";
    device.dtid = @"<Sami Phone Device Type ID>";
     
    [api addDeviceWithCompletionBlock:device completionHandler:^(SamiDeviceEnvelope *output, NSError *error) {
        if (!error) {
            NSLog("@Registered Device with ID: %@", output.data._id);
        } else {
            NSLog(@"Error %@", error);
        }
    }];
}
~~~

## Post messages

Once the device is created and the deviceId is known, the iOS app can post messages. The message data, needs to match the Manifest information. The message data needs to be marshaled within NSDictionary.

**SamiAddMessageViewController.m**

~~~
- (IBAction)addMessage:(id)sender {
    NSString *deviceId = [SamiUserSession sharedInstance].currentDeviceId;
    NSString* authorizationHeader = [SamiUserSession sharedInstance].bearerToken;
    
    SamiMessagesApi * api2 = [SamiMessagesApi apiWithHeader:authorizationHeader key:OAUTH_AUTHORIZATION_HEADER];
    
    SamiMessage *message = [[SamiMessage alloc] init];
    message.sdid = deviceId;
    message.data = @{ @"numberOfSteps": @([self.stepsField.text integerValue]),
                      @"floorsAscended": @([self.floorsAscField.text integerValue]),
                      @"floorsDescended": @([self.floorsDescField.text integerValue]),
                      @"distance": @([self.distanceField.text floatValue])};
    
    [api2 postMessageWithCompletionBlock:message completionHandler:^(SamiMessageIDEnvelope *output, NSError *error) {
        if (!error) {
            UIAlertView *alert = [[UIAlertView alloc] initWithTitle:@"Success"
                                                            message:[@"Message added " stringByAppendingString:output.data.mid]
                                                           delegate:nil
                                                  cancelButtonTitle:@"OK"
                                                  otherButtonTitles:nil];
            [alert show];
        } else {
            NSLog(@"%@", error);
            
            UIAlertView *alert = [[UIAlertView alloc] initWithTitle:@"Error"
                                                            message:error.localizedDescription
                                                           delegate:nil
                                                  cancelButtonTitle:@"OK"
                                                  otherButtonTitles:nil];
            [alert show];
        }
    }];
}

~~~

In the code block above, the data of the message represents the pedometer data that has 4 numerical fields:

- numberOfSteps
- floorsAscended
- floorsDescended
- distance

Other API calls can be similarly invoked.

## Logging out

The iOS app can log out the user (invalidate the access token) by accessing this endpoint.

~~~
https://accounts.samsungsami.io/logout?redirect_uri 
~~~

## License and copyright

Licensed under the APACHE license. See [LICENSE.](https://github.com/samsungsamiio/sami-ios/blob/master/LICENSE)

Copyright (c) Samsung Electronics Co., Ltd.
