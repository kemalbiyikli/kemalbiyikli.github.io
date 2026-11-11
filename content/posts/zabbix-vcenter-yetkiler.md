---
title: "Zabbix ile vCenter Monitoring: Hangi Kullanıcı Hangi Gruplarda Olmalı?"
date: 2026-07-15T00:00:00
draft: true
tags:
  - VMware
  - vCenter
  - Zabbix
  - Monitoring
slug: zabbix-ile-vcenter-monitoring-hangi-kullanici-hangi-gruplarda-olmali
---
Zabbix üzerinden bir vCenter ortamını izlerken kullanılan servis hesabına ne kadar yetki verileceği sık karışan bir konu. “Read-Only rol yeterli” diye başlıyoruz, sonra bazı item’lar sürekli hata veriyor, çaresizlikle Administrator veriyoruz ve her şey düzeliyor ki bu da izleme hesabı için gereğinden fazla yetki demek. Peki aslında ihtiyacımız olan yetkiler neler?

## Neden Read Only yetmiyor?

vCenter’da aslında birbirinden bağımsız iki ayrı yetkilendirme sistemi var:

1. **Global Permissions / Roles** — Inventory kısmını, yani VM, host, cluster, datastore, folder gibi **inventory objelerine** erişimi kontrol eder.
2. **SSO domain grupları** **WAMI** — Appliance’ın kendi iç servislerini; health, sertifikalar, backup durumu gibi bilgileri kontrol eden, tamamen ayrı bir yetki katmanı.

VMware’in kendi dokümantasyonu bunu açıkça belirtiyor: vCenter üzerinde Global Permissions'a Read Only yetkisi envanter'e tam okuma izni vermesine rağmen appliance health verisine erişim sağlamıyor. 

## Gereken üç grup

### 1. Global Permissions → Read-Only rolü

**Nereye bakar:** vCenter envanteri; VM’ler, ESXi host’lar, cluster’lar, datastore’lar, resource pool’lar, folder’lar.

**Nasıl atanır:** Administration → Access Control → Global Permissions → kullanıcıyı/grubu ekle → Role: **Read-only** → Propagate to children işaretli.

**Zabbix’e sağladığı:** VM ve host envanteri, CPU/memory/disk/network metrikleri, folder, datastore kapasite bilgisi vb.

### 2. SSO Group: SystemConfiguration.ReadOnly

**Nereye bakar:** Appliance Management API (VAMI’nin arkasındaki `/api/appliance/...` endpoint’leri). vCenter servislerinin durumu (vpxd, vpostgres, vsphere-ui vb.), appliance sertifika süreleri, backup job geçmişi, disk/CPU/memory kullanımı — appliance’ın **kendi işletim sistemi seviyesindeki** sağlık durumu.

**Nasıl atanır:** Administration → Single Sign-On → Users and Groups → Groups sekmesi → domain seç → `SystemConfiguration.ReadOnly` grubunu bul → kullanıcıyı Members olarak ekle.

**Zabbix’e sağladığı:** “Overall Health VC State” gibi appliance health item’ları. Bu grup olmadan, Global Permissions Read-Only ne kadar geniş verilirse verilsin, appliance health sorgusu `403 UNAUTHORIZED` döner — çünkü sorulan kaynak farklı bir yetki alanında.

### 3. SSO Group: vStatsGroup

**Nereye bakar:** vCenter’ın performans istatistik servisi (vStats). Detaylı performance counter verilerine erişim bu grup üzerinden kontrol edilir.

**Nasıl atanır:** Aynı ekrandan — Groups sekmesinde `vStatsGroup`‘u bulup kullanıcıyı Members olarak ekleyin.

**Zabbix’e sağladığı:** Zabbix’in resmi VMware entegrasyon dokümantasyonu, ayrı bir monitoring hesabı kullanılacaksa bu hesabın hem SystemConfiguration.ReadOnly hem de vStatsGroup üyesi olması gerektiğini açıkça belirtiyor — ikisi birlikte istendiği için performans verisi eksiksiz toplanabiliyor.

## Özet tablo


| Grup / Rol | Yetki katmanı | Kapsadığı veri | Atandığı yer |
| ------------------------------ | ------------------ | -------------------------------------------------------- | ------------------------------------------------ |
| Global Permissions → Read-Only | vSphere RBAC | VM, host, datastore, cluster envanteri ve sayaçları | Administration → Global Permissions |
| SystemConfiguration.ReadOnly | SSO built-in group | Appliance health, servis durumları, sertifikalar, backup | Administration → SSO → Users and Groups → Groups |
| vStatsGroup | SSO built-in group | Detaylı performans istatistikleri (vStats) | Administration → SSO → Users and Groups → Groups |


## Pratik not

Bu üç atama birbirini geçersiz kılmaz, üst üste eklenir. Bir izleme hesabı için doğru pratik:

- Administrator rolü **vermeyin** — gereğinden fazla yetki, güvenlik açısından gereksiz risk.
- Yukarıdaki üç grubu/rolü ayrı ayrı atayın.
- SSO grup üyelikleri Global Permissions ekranında **görünmez** — bunları doğrulamak için tek yer Administration → SSO → Users and Groups → Groups → ilgili grup → Members sekmesi.
- SSO grup üyeliği bazı vCenter sürümlerinde hemen değil, birkaç dakika içinde propagate olabiliyor; test etmeden önce biraz bekleyin.

Bu üç parça bir araya geldiğinde Zabbix’in VMware template’i (inventory + appliance health + performans istatistikleri) eksiksiz ve en az yetki prensibiyle çalışır.

## Kaynaklar

Aşağıdaki resmi Broadcom/VMware TechDocs sayfaları, yazıdaki bilgilerin dayandığı birincil kaynaklardır (vSphere 8.0 dokümantasyon seti; sürüm seçici üzerinden 7.0/9.0 gibi diğer sürümlere de geçilebilir):

1. **Groups in the vCenter Single Sign-On Domain** — vsphere.local domain’indeki tüm önceden tanımlı (built-in) grupları ve her birinin ne işe yaradığını listeleyen ana referans tablo. `SystemConfiguration.ReadOnly` ve `vStatsGroup` satırları burada resmi olarak tanımlanıyor. `https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-sdks-tools/8-0/groups-in-the-vsphere-local-domain.html` İlgili tablo satırları (sayfadan birebir):

  *SystemConfiguration.ReadOnly*: “Members of this group can access vCenter Server Appliance read-only operations under Appliance Management.”

   *vStatsGroup*: “This group is used for vStats collection.”

   *SystemConfiguration.Administrators*: “Members of the SystemConfiguration.Administrators group can view and manage the system configuration in the vCenter Server Management Interface running on port 5480…”
2. **Add a User to the SystemConfiguration.BashShellAdministrators Group** — SystemConfiguration.* grup ailesine üye ekleme adımlarının (Administration → Single Sign On → Users and Groups → Groups → ilgili grup → Edit) uygulamalı örneği. Aynı adımlar SystemConfiguration.ReadOnly için de geçerli. `https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere/8-0/vcenter-configuration/configuring-vcenter-server-using-the-vsphere-client/add-a-user-to-the-systemconfiguration-bashshelladministrators-group.html`
3. **Using vCenter Server Global Permissions** — Global Permissions’ın kapsamının vCenter inventory hiyerarşisi (VM, host, datastore, tag, content library) ile sınırlı olduğunu, ayrı bir global root objesi üzerinden uygulandığını anlatan bölüm. `https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere/8-0/vsphere-security/vsphere-permissions-and-user-management-tasks/global-permissions.html`
4. **Understanding Authorization in vSphere** — Global Permissions ile SSO grup üyeliğinin iki ayrı yetkilendirme modeli olduğunu, birinin inventory objelerini, diğerinin SSO tarafından yönetilen servisleri kapsadığını açıklayan kavramsal bölüm. `https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere/8-0/vsphere-security/vsphere-permissions-and-user-management-tasks/understanding-authorization-in-vsphere.html`
5. **Zabbix VMware entegrasyon dokümantasyonu** — ayrı bir monitoring hesabı kullanılacaksa hesabın `SystemConfiguration.ReadOnly` ve `vStatsGroup` gruplarının üyesi olması gerektiğini belirten resmi Zabbix template dokümantasyonu. `https://www.zabbix.com/integrations/vmware`

> Not: Broadcom TechDocs URL’leri sürüm bazlı (`/8-0/`, `/7-0/`, `/9-0/` gibi) yapılandırılıyor; kendi vCenter sürümünüze göre sayfadaki sürüm seçiciden doğru dokümana geçebilirsiniz.

