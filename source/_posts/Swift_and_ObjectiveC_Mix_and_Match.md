title: Swift and Objective-C (Mix and Match)
date: 2016-07-25 17:32:53
categories: Development
tags: [Swift]

---

Swift 与 Objective-C 混编 （to be continued）

### Code 直接混编 + Framework 间接混编

-----

## < Code 直接混编 >

* #### 在 Objective-C code 中使用 Swift code

	Objective-C project: `Project_C`
	PRODUCT_MODULE_NAME: `Project_C_Product`

	1. in Project_C's build settings, set up `"Embedded Content Contains Swift Code"` to `"YES"`.
	2. in Project_C's build settings, set up `"Objective-C Generated Interface Header Name"` to `"Project_C_Product-Swift.h"`.
	3. Swift code should use `@objc` to mark what you want to expose to Objective-C code.
	4. in Objective-C code, `#import "Project_C_Product-Swift.h"`.


<!--more-->


* #### 在 Swift code 中使用 Objective-C code

	Swift project: `Project_S`
	PRODUCT_MODULE_NAME: `Project_S_Product`

	1. in Project_S's build settings, set up `"Objective-C Bridging Header"` to `"Project_S_Product-Bridging-Header.h"`.
	2. in `"Project_S_Product-Bridging-Header.h"`, `#import` every Objective-C header you want to expose to Swift code.
	3. in Swift code, *NO* need to `#import "Project_C_Product-Bridging-Header.h"`.

## < Framework 间接混编 >

* #### 创建 Swift framework 给 Objective-C code 使用

  framework product: `Framework_S`

  **framework's side:**
  1. in Framework_S's build settings, set up `"Objective-C Generated Interface Header Name"` to `"Framework_S-Swift.h"`.
  2. use `@objc` to mark what you want to expose to Objective-C code.
  3. after Framework_S is producted, its umbrella will include the `"Framework_S-Swift.h"` file.

  **consumer's side:**
  1. in build settings, set up `"Embedded Content Contains Swift Code"` to `"YES"`.
  2. `@import Framework_S`.

* #### 创建 Objective-C framework 给 Swift code 使用

  framework product: `Framework_C`

  **framework's side:** nothing

  **consumer's side:** `import Framework_C`

-----

#### [Apple 官方文档] [1]

-----

[1]: https://developer.apple.com/library/ios/documentation/Swift/Conceptual/BuildingCocoaApps/MixandMatch.html
