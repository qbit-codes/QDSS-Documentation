# QBitwise Biyometrik İmza Uygulaması Hakkında
Bu uygulama yüklü olduğu android cihazda .pdf dosylarını görüntülüyebilir, belirlenmiş imza alanlarını imzalyabilir veya yeni bir imza alanı oluşturabilir. Bir pdf görüntüleyici gibi davrandığı için dosya dizininden seçilen bir pdf'i imzalayabilirsiniz. Harici bir uygulamadan İmza uygulamasını çağırabilirsiniz. İmzalama işlemi tamamlandığında imzalı pdf'in path'i çağıran uygulamaya parametre olarak iletilir.

### <a id="qs-add-sdk"></a>İmza Uygulaması Kurulumu
Uygulama cihaz kurulduktan sonra izinlerin verilmesi, PublicKey ve İmza Sertifikasının yüklenmesi gerekmektedir. Aksi taktirde İmza Uygulaması kulurulumu tamamlanmadı hatası döner.

Ayarlar ekranın yönetici parolası: 000000 (6 tane 0) olarak atanmıştır.

```text
1-) İzinlerin Verilmesi
2-) PublicKey Yüklenmesi
3-) İmza Sertifikasının Yüklenmesi
4-) İmza sertifika parolasının girilmesi.
```

### <a id="qs-add-sdk"></a>Harici bir uygulamadan erişim yönetemleri

Uygulamanızın bir FileProvider'ı olduğundan emin olun. Örnek FileProvider aşağıdaki gibidir.
```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.FileProvider"
    android:exported="false"
    android:grantUriPermissions="true">

    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_provider_paths" />

</provider>
```

xml/file_provider_paths:

```xml
<paths>
    <files-path
        name="files"
        path="." />
    <cache-path
        name="cache"
        path="." />
</paths>
```
### <a id="qs-add-sdk"></a>Biyometrik İmza Uygulaması Activity ve PackageName
Biyometrik İmza Uygulaması'nı çağırmak için uygulamanın Activity ve Packagename'ine ihtiyacınız olacak. Bu bilgileri string.xml dosyanıza ekleyin.

```xml
<string name="digital_signature_app_package">com.qbitwise.digitalsignature</string>
<string name="digital_signature_app_sign_activity">com.qbitwise.digitalsignature.presentation.SignActivity</string>
```

### <a id="qs-add-sdk"></a>Biyometrik İmza Uygulaması'nı Başlatmak
Intent.ACTION_VIEW tipinde bir android Intent objesi oluşturun. Biyometrik İmza Uygulaması başlatma ayarlarını putExtra methodu ile intent'e ekleyin. Ardından StartActivityForResult yönetimi ile intent'i başlatın.

```kotlin
    private fun launchDigitalSignature(uri: Uri) {
    val intent = Intent(Intent.ACTION_VIEW)

    intent.data = uri
    intent.component = ComponentName(
        getString(R.string.digital_signature_app_package),
        getString(R.string.digital_signature_app_sign_activity)
    )

    intent.putExtra("output_file_name", "signed_file.pdf")
    intent.putExtra("click_to_sign", true)
    intent.putExtra("verify_sign_with_id", true)
    intent.putExtra("public_key","PUBLIC_KEY_BASE64")
    intent.putExtra("signature_certificate","SIGNATURE_CERTIFICATE_BASE64")
    intent.putExtra("signature_certificate_password","SIGNATURE_CERTIFICATE_PASSWORD")

    intent.flags = Intent.FLAG_GRANT_READ_URI_PERMISSION or Intent.FLAG_GRANT_WRITE_URI_PERMISSION
    intent.setDataAndType(uri, "application/pdf")

    try {
        signatureAppLauncher.launch(intent)
    } catch (e: ActivityNotFoundException) {
        Log.e("ActivityNotFoundException ", "İmza uygulaması yüklü değil!")
    }
}
```

### <a id="qs-add-sdk"></a>İmzalanan Pdf'i Alma
StartActivityForResult yönetimi ile çağırılan intent sonlandığında aşağıdaki gibi imzalanan pdf'in path'i yakalayabilirsiniz.

```kotlin
    private var signatureAppLauncher =
    registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result ->
        if (result.resultCode == Activity.RESULT_OK) {
            val data: Intent? = result.data
            data?.let {
                it.getStringExtra("output_file")?.let { file ->
                    Log.d("Output file path: ", file)
                } ?: kotlin.run {
                    Log.e("Output file path: ", "NULL")
                }
            }
        }
    }
```

### <a id="qs-add-sdk"></a>Biyometrik verilerin şifrelenmesi için RSA Key Pair oluşturma
Her imzanın biyometrik verileri pdf'te imza alananına gömülür. Bu datalar her ne kadar özel tool'lar kullanmadan erişilemez olsa da encrypt edilerek kaydedilir. Verilerin encrypt edilirken asimetrik şifreleme yöntemi kullanılır. Asimetrik şifrelemede kullanılacak public ve private keyleri aşağıdaki adımları izleyerek oluşturulabilirsiniz.

Private Key
```text
openssl genrsa -out private.pem 1024
```

Public Key
```text
openssl rsa -in private.pem -outform PEM -pubout -out public.pem
```

### <a id="qs-add-sdk"></a>İmza Sertifikası Oluşturma

1-) Generate 2048-bit RSA private key:
```text
openssl genrsa -out key.pem 2048
```

2-) Generate a Certificate Signing Request:
```text
openssl req -new -sha256 -key key.pem -out csr.csr
```

3-) Generate a self-signed x509 certificate suitable for use on web servers.
```text
openssl req -x509 -sha256 -days 365 -key key.pem -in csr.csr -out certificate.pem
```

4-) Create SSL identity file in PKCS12 as mentioned here
```text
openssl pkcs12 -export -out signature-certificate.p12 -inkey key.pem -in certificate.pem
```

### <a id="qs-add-sdk"></a>PublicKey ve İmza sertifikasını base64'e dönüştürme
p12_to_base64.jar'ı indirin public.pem ve signature-certificate.p12 ile aynı klasör içinde aşağıdaki java komutunu çağırın. Klasör içerisinde public-key-base64.txt ve signature-certificate-base64.txt dosyaları oluşturulacaktır.
```text
java -jar p12_to_base64.jar p12-path=signature-certificate.p12 public-key-path=public.pem
```
