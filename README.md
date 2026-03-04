# UEFI Tabanlı Boot Seviyesi Manipülasyon ve Müdahale Analizi

Bu doküman, modern anti-cheat sistemlerini (Vanguard, EAC, BattlEye) atlatmak amacıyla geliştirilen ve son dönemde FiveM, Valorant, CS2 gibi platformlarda yaygınlaşan EFI/UEFI tabanlı müdahale yöntemlerinin teknik çalışma prensiplerini ve tespit senaryolarını incelemektedir.

---

## Genel Bakış

Geleneksel manipülasyon yazılımları, işletim sistemi üzerinde çalıştırılabilir dosyalar (.exe, .dll) olarak faaliyet gösterir. Ancak modern çekirdek seviyesi (kernel-mode) anti-cheat mekanizmaları, Windows çekirdeği yüklendiği andan itibaren sistemi denetlemeye başlar. Geliştiriciler, bu denetim mekanizmalarından kaçınmak amacıyla müdahale yazılımlarını işletim sistemi henüz yüklenmeden, Boot Loader aşamasında devreye sokarak tespit yüzeyini (detection surface) minimize etmeyi hedeflemektedir.

## Çalışma Mantığı

Modern sistemlerde BIOS'un yerini alan UEFI (Unified Extensible Firmware Interface), işletim sisteminden önce çalışan ve donanım başlatma süreçlerini yöneten bir arabirimdir. UEFI tabanlı müdahale süreci genel olarak aşağıdaki adımları izler:

1. UEFI Firmware Başlatma: Sistem donanımını hazırlar ve önyükleme önceliğini belirler.
2. EFI Application Injection: Standart Windows Boot Manager yüklenmeden önce, özel olarak hazırlanmış bir .efi dosyası (genellikle harici bir bellek veya manipüle edilmiş bir disk bölümü aracılığıyla) sisteme enjekte edilir.
3. Kernel Hooking: EFI uygulaması, Windows Çekirdeği (ntoskrnl.exe) belleğe yüklenirken araya girer (hooking). Çekirdek tam yetki kazanmadan önce ilgili kod belleğe yerleşir.
4. Runtime Manipulation: İşletim sistemi açıldığında, manipülasyon kodu çekirdeğin daha alt bir katmanında (Pre-Kernel / Ring -1) çalıştığı için standart sürücü taramaları tarafından tespit edilemez.

### Temel Teknik Özellikler

* Dosyasız Çalışma (Fileless): Sistem başlatıldıktan sonra disk üzerinde aktif bir dosya izi bırakmaz; tüm operasyon RAM üzerinde yürütülür.
* Yüksek Yetki Konumu: Çekirdek yüklenmeden önce devreye girdiği için, Sürücü İmza Zorlaması (DSE - Driver Signature Enforcement) gibi işletim sistemi güvenlik protokollerini atlatabilir.

---

## Kritik Uyarı ve Teknik Not

Bu yöntem, doğrudan bir Windows çekirdek sürücüsü (kernel driver) olarak değil, çekirdeğin alt katmanında (pre-OS environment) gerçekleşen bir manipülasyon olarak değerlendirilmelidir. Bu durum, söz konusu yazılımları basit bir oyun modifikasyonundan çıkarıp, teknik terminolojide Bootkit (Rootkit türevi) seviyesine taşımaktadır.

---

## Tespit Metodolojisi

EFI tabanlı yöntemler tamamen görünmez değildir. Bu dokümanda sunulan tespit mantığı, "Katmanlar Arası Tutarsızlık Analizi" ve "Önyükleme Yapılandırma (BCD) Denetimi" prensiplerine dayanmaktadır.

### 1. Katmanlar Arası Tutarsızlık (Secure Boot Senkronizasyon Bozukluğu)

EFI tabanlı bootkitler, çekirdeğin altında çalıştıkları için standart taramalardan gizlenebilirler. Ancak bu sistemlerin en belirgin zayıf noktası, Güvenli Önyükleme (Secure Boot) simülasyonudur. Bir EFI manipülasyonunun çalışabilmesi için donanımsal düzeyde Secure Boot'un kapalı olması gerekir. Buna karşın, hedef yazılımı atlatabilmek için işletim sistemini bu özelliğin açık olduğuna ikna etmek zorundadır.

Bu senkronizasyon bozukluğu iki farklı veri noktasının karşılaştırılmasıyla tespit edilebilir:

* Yazılımsal Beyan (OS Registry): Windows, Secure Boot durumunu kayıt defterinde (HKLM) tutar. Manipülasyon yazılımları bu değeri 1 (Açık) olarak değiştirerek işletim sistemini yanıltır.
* Donanımsal Gerçeklik (NVRAM): Gerçek Secure Boot durumu, anakartın NVRAM değişkenlerinde muhafaza edilir. Doğrudan donanım katmanına sorgu gönderilerek gerçek durum doğrulanabilir.

### 2. BCD (Boot Configuration Data) Manipülasyon Analizi

Bazı EFI müdahale yazılımları, sistem başlangıcında kendi kodlarını çalıştırabilmek için Windows Önyükleme Yöneticisi (Windows Boot Manager) yapılandırmasına veya doğrudan NVRAM boot girdilerine müdahale eder. Standart bir Windows kurulumunda önyükleme yolu genellikle `\EFI\MICROSOFT\BOOT\BOOTMGFW.EFI` şeklindedir. 

Müdahale yazılımları bu yolu değiştirerek farklı bir disk bölümündeki (veya USB bellekteki) sahte bir `.efi` dosyasına yönlendirme yapabilir. Bu anormallikler `bcdedit /enum all` ve özellikle doğrudan UEFI Firmware (NVRAM) boot girdilerini listeleyen `bcdedit /enum firmware` komutları incelenerek tespit edilebilir. Bu taramalar sayesinde standart dışı boot uygulamaları ve `nointegritychecks`, `testsigning` gibi güvenlik atlatma parametreleri ortaya çıkarılır.

---

## Vaka Analizi: Vionex.cc Örneği

Yapılan inceleme kapsamında, vionex.cc sitesi üzerinden temin edilen bir müdahale yazılımı teknik analize tabi tutulmuştur. Kullanıcıya "Secure Boot Bypass" sağladığı iddiasıyla sistemi yeniden başlatmaya zorlayan bu tür yazılımların aşağıdaki adımları izlediği doğrulanmıştır:

* Sistemi özel bir EFI Shell veya manipüle edilmiş Firmware BCD kaydı üzerinden başlatmak.
* Windows Çekirdeği yüklenmeden önce PolicyPublisher değerini manipüle ederek yetki kazanmak.
* Hedef yazılıma "Sistem Güvenli" (Secure Boot: ON) sinyali gönderirken, donanım katmanında ilgili korumaları devre dışı bırakmak.

### Kanıt ve Uygulamalı Gösterim

Bahsi geçen EFI tabanlı çözümün çalışma mekanizmasını, sistem üzerindeki etkilerini ve Kayıt Defteri ile NVRAM arasındaki senkronizasyon çelişkisini aşağıdaki teknik analiz videosunda inceleyebilirsiniz:

Test ve Yakalama Videosu: [Vionex EFI Cheat](https://vimeo.com/1170353586)

---

## Örnek Tespit Betiği (PowerShell)

Aşağıdaki PowerShell betiği, belirtilen katmanlar arası tutarsızlığı, BCD yapılandırmasındaki ve Firmware NVRAM kayıtlarındaki anormallikleri tespit etmek amacıyla hazırlanmıştır. Betiğin yönetici ayrıcalıklarıyla (Administrator) çalıştırılması gerekmektedir.

```powershell
Write-Host "--------------------------------------------------------" -ForegroundColor Cyan
Write-Host "Majdev: Starting EFI Spoofer / Bootkit Scan..." -ForegroundColor Cyan
Write-Host "--------------------------------------------------------" -ForegroundColor Cyan

$isCheating = $false
$reason = ""

$regPath = "HKLM:\SYSTEM\CurrentControlSet\Control\SecureBoot\State"
$regState = $null
$regPublisher = $null

try {
    $regState = (Get-ItemProperty -Path $regPath -Name "UEFISecureBootEnabled" -ErrorAction Stop).UEFISecureBootEnabled
} catch {
    $regState = 0
}

try {
    $regPublisher = (Get-ItemProperty -Path $regPath -Name "PolicyPublisher" -ErrorAction SilentlyContinue).PolicyPublisher
} catch {
}

$firmwareState = $null
try {
    $firmwareState = Confirm-SecureBootUEFI
} catch {
    $firmwareState = $false
}

Write-Host "Registry (OS) Declaration : " -NoNewline
if ($regState -eq 1) { Write-Host "ON" -ForegroundColor Green } else { Write-Host "OFF" -ForegroundColor Red }

Write-Host "Motherboard (Hardware) Declaration : " -NoNewline
if ($firmwareState) { Write-Host "ON" -ForegroundColor Green } else { Write-Host "OFF" -ForegroundColor Red }

if ($regState -eq 1 -and $firmwareState -eq $false) {
    $isCheating = $true
    $reason = "Critical Mismatch! Registry shows Secure Boot 'ON' while hardware reports 'OFF'."
}

if ($regPublisher -eq "{77fa9abd-0359-4d32-bd60-28f4e78f784b}") {
    $isCheating = $true
    $reason += " Additionally, a fake PolicyPublisher GUID signature (77fa9abd...) was detected."
}

if ($isCheating) {
    Write-Host "[!] RESULT: DETECTION FOUND (EFI BOOTED)" -ForegroundColor Red -BackgroundColor Black
}
else {
    
}

Write-Host "--------------------------------------------------------" -ForegroundColor Cyan
Write-Host "`nYou can scroll up to review the results." -ForegroundColor Gray
Read-Host "Press Enter to close the window"
