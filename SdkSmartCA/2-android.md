

# Tích hợp Android
:::caution Yêu cầu
- Yêu cầu Android SDK >= 6.0 (API level 23).
:::

## Bước 1: Tải SDK và cấu hình Project

- Tải về bộ tích hợp SDK tại https://github.com/VNPT-SmartCA/vnpt_smartca_sdk_android và giải nén ra thư mục.
- Sau khi giải nén, copy các file aar, thư mục repo vào thư mục **libs** trong **app** (1)
-  Thêm thông tin cấu hình vào file app/settings.gradle như dưới:
```` kotlin
pluginManagement {  
  repositories {  
  gradlePluginPortal()  
        google()  
        mavenCentral()  
    }  
}  
  
String storageUrl = System.env.FLUTTER_STORAGE_BASE_URL ?: "https://storage.googleapis.com"  
  
dependencyResolutionManagement {  
  repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)  
    repositories {  
  google()  
        mavenCentral()  
        maven {  
        //Đường dẫn thư mục chứa **repo** ở bước 1
  url 'app/libs/repo'  
  }  
  // bỏ qua nếu project phát triển bằng Flutter
  maven {  
  url "$storageUrl/download.flutter.io"  
  }  
  maven { url "https://jitpack.io" }  
  
  jcenter()  
    }  
}  
````
- Thêm các implementation vào app/build.gradle
````dependencies {
//..
implementation 'com.vnpt.smartca.module.vnpt_smartca_module:flutter_release:1.0'  
//Đường dẫn tới các file aar
implementation files('libs/vnpt_smartca_sdk_lib-release.aar')  
implementation files('libs/ekyc_sdk-release-v3.2.7.aar')  
implementation files('libs/eContract-v3.1.0.aar'
 }
````

- Thêm FlutterActivity trong file **AndroidManifest.xml** như sau:

```` java
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-feature android:name="android.hardware.camera" />
    //......
    <application
        //......
        
        // bỏ qua nếu project phát triển bằng Flutter
        <activity android:name="io.flutter.embedding.android.FlutterActivity"
            android:theme="@style/Theme.Smartca_android_example"
            android:configChanges="orientation|keyboardHidden|keyboard|screenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
            android:hardwareAccelerated="true"
            android:windowSoftInputMode="adjustResize"/>

    </application>
````
- Bổ sung thuộc tính dưới đây vào file **gradle.properties**

```` java
    android.enableJetifier=true
````
- Bổ sung thuộc tính dưới đây vào file **proguard-rules.pro**

```` java
    -keep class ai.icenter.face3d.native_lib.Face3DConfig { *; }
````


## Bước 2: Khởi tạo SDK tại nơi muốn bắt đầu kết nối

- Thêm đoạn code dưới đây tại **Activity** muốn kết nối với SDK trước khi sử dụng các chức năng:
 ```` java
import com.vnpt.smartca.ConfigSDK  
import com.vnpt.smartca.SmartCAEnvironment  
import com.vnpt.smartca.SmartCALanguage  
import com.vnpt.smartca.SmartCAResultCode  
import com.vnpt.smartca.VNPTSmartCASDK

class MainActivity : AppCompatActivity() {
	var VNPTSmartCA = VNPTSmartCASDK()
	lateinit var editTextTrans: EditText
    override fun onCreate(savedInstanceState: Bundle?) {  
    super.onCreate(savedInstanceState)  
  
    setContentView(R.layout.activity_main)  
    val btnActiveAccount = findViewById<Button>(R.id.btnActiveAccount)  
    editTextTrans = findViewById(R.id.plain_text_input)  
  
    val btnConfirmTrans = findViewById<Button>(R.id.btnConfirmTrans)  
    val bntMainInfo = findViewById<Button>(R.id.btnMainInfo)  
  
    val config = ConfigSDK()  
    config.context = this  
    //clientId của đối tác được VNPTSmartCA cung cấp khi yêu cầu tích hợp
	  config.partnerId = "xxxxx"  
	//Cấu hình môi trường Dev-test hay Production cùa SmartCA
	  config.environment = SmartCAEnvironment.DEMO_ENV  
    //Cấu hình ngôn ngữ app (vi/en)
	  config.lang = SmartCALanguage.VI
    //Cấu hình chọn project là Flutter hay không
	  config.isFlutter = false  
	  VNPTSmartCA.initSDK(config)  
  //..
}
````

## Bước 3: Sử dụng các hàm chính

- Kích hoạt tài khoản /lấy thông tin xác thực người dùng (accessToken & credentiald)
- Quản lý tài khoản
- Xác nhận giao dịch ký số
- Hủy kết nối SDK
````
### 📦 Hàm quản lý tài khoản  

- Đối tác sẽ **Quản lý tài khoản** thông qua hàm getMainInfo. Nếu chưa Login thì sẽ show màn hình để Login, nếu đã login thì sẽ show SDK.

```` kotlin
 fun getMainInfo() {  
    try {  
        VNPTSmartCA.getMainInfo { result ->  
  when (result.status) {  
                SmartCAResultCode.SUCCESS_CODE -> {  
                    // Xử lý khi confirm thành công  
  }  
  
                else -> {  
                    // Xử lý khi confirm thất bại  
  }  
            }  
        }  
  } catch (ex: java.lang.Exception) {  
        throw ex;  
  }  
}
````
### 📦 Hàm lấy accessToken và credentialId của người dùng

SDK sẽ thực hiện kiểm tra trạng thái tài khoản và chứng thư của khách hàng như: đã kích hoạt hay chưa, chứng thư hợp lệ hay không, tự động làm mới token khi hết hạn,.... Thành công SDK sẽ trả về **accessToken** và **credentialId** của người dùng

```` kotlin
private fun getAuthentication() {  
    try {  
        VNPTSmartCA.getAuthentication { result ->  
  when (result.status) {  
                SmartCAResultCode.SUCCESS_CODE -> {  
                    val obj: CallbackResult = Json.decodeFromString(  
                        CallbackResult.serializer(), result.data.toString()  
                    )  
                    // SDK trả lại token, credential của khách hàng  
 // Đối tác tạo transaction cho khách hàng để lấy transId, sau đó gọi getWaitingTransaction  val token = obj.accessToken  
  val credentialId = obj.credentialId  
  
  val builder = AlertDialog.Builder(this)  
                    builder.setTitle("Xác thực thành công")  
                    builder.setMessage("CredentialId: $credentialId;\nAccessToken: $token")  
                    builder.setPositiveButton(  
                        "Close"  
  ) { dialog, _ -> dialog.dismiss() }  
  builder.show()  
                }  
  
                else -> {  
                    // Xử lý lỗi  
  val builder = AlertDialog.Builder(this)  
                    builder.setTitle("Thông báo")  
                    builder.setMessage("status: ${result.status}; statusDesc: ${result.statusDesc}")  
                    builder.setPositiveButton(  
                        "Close"  
  ) { dialog, _ -> dialog.dismiss() }  
  builder.show()  
                }  
            }  
        }  
  } catch (ex: Exception) {  
        throw ex;  
  }  
}
````
### 📦 Hàm xác nhận giao dịch 

- Sau khi lấy được **accessToken và credentialId** của người dùng từ **getAuthentication**, từ phía **Backend** của Đối tác tạo giao dịch ký số cho khách hàng, lấy **tranId** sau đó gọi hàm xác nhận ký số.

```` java
private fun getWaitingTransaction(transId: String) {  
    try {  
        if (transId.isNullOrEmpty()) {  
            editTextTrans.error = "Vui lòng điền Id giao dịch";  
 return  }  
  
        VNPTSmartCA.getWaitingTransaction(transId) { result ->  
  val builder = AlertDialog.Builder(this)  
            builder.setTitle("Thông báo")  
            builder.setMessage("status: ${result.status}; statusDesc: ${result.statusDesc}")  
            builder.setPositiveButton(  
                "Close"  
  ) { dialog, _ -> dialog.dismiss() }  
  builder.show()  
  
            when (result.status) {  
                SmartCAResultCode.SUCCESS_CODE -> {  
                    // Xử lý khi thành công  
  }  
  
                else -> {  
                    // Xử lý khi thất bại  
  }  
            }  
  
        }  
  } catch (ex: Exception) {  
        throw ex;  
  }  
}
### 📦 Hàm hủy kết nối SDK
```` kotlin
override fun onDestroy() {
    VNPTSmartCA.destroySDK()
    super.onDestroy()
}
````