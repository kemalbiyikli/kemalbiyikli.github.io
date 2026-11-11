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

**Nereye bakar:** Appliance Management API (VAMI). vCenter servislerinin durumu (vpxd, vpostgres, vsphere-ui vb.), appliance sertifika süreleri, Disk/CPU/Memory kullanımı, appliance’ın **kendi işletim sistemi seviyesindeki** sağlık durumu.

**Nasıl atanır:** Administration → Single Sign-On → Users and Groups → Groups sekmesi → domain seç → `SystemConfiguration.ReadOnly` grubunu bul → kullanıcıyı Members olarak ekle.

**Zabbix’e sağladığı:** “Overall Health VC State” gibi appliance health item’ları. Bu grup olmadan, Global Permissions Read-Only ne kadar geniş verilirse verilsin, appliance health sorgusuna yanıt alınamaz.

### 3. SSO Group: vStatsGroup

**Nereye bakar:** vCenter’ın performans istatistik servisi (vStats). Detaylı performance counter verilerine erişim bu grup üzerinden kontrol edilir.

**Nasıl atanır:** Aynı ekrandan — Groups sekmesinde `vStatsGroup`‘u bulup kullanıcıyı Members olarak ekleyin.

**Zabbix’e sağladığı:** Zabbix’in resmi VMware entegrasyon dokümantasyonu, ayrı bir monitoring hesabı kullanılacaksa bu hesabın hem SystemConfiguration.ReadOnly hem de vStatsGroup üyesi olması gerektiğini  belirtiyor, ikisi birlikte sağlandığında performans verisinin eksiksiz toplanabileceği belirtiliyor.

## Özet tablo


| Grup / Rol | Yetki katmanı | Kapsadığı veri | Atandığı yer |
| ------------------------------ | ------------------ | ---------------------------------------------------- | ------------------------------------------------ |
| Global Permissions → Read-Only | vSphere RBAC | VM, Host, Datastore, Cluster gibi envanter item'ları | Administration → Global Permissions |
| SystemConfiguration.ReadOnly | SSO built-in group | Appliance health, servis durumları, sertifikalar, | Administration → SSO → Users and Groups → Groups |
| vStatsGroup | SSO built-in group | Detaylı performans istatistikleri (vStats) | Administration → SSO → Users and Groups → Groups |


## Notlar

Bu üç atama üst üste eklenir ve farklı amaçlara hizmet ederler.

izleme için **Administrator** rolü vermeyin gereğinden fazla yetki her zaman risktir.

SSO grup üyelikleri Global Permissions ekranında **görünmez** — bunları doğrulamak için tek yer Administration → SSO → Users and Groups → Groups → ilgili grup → Members sekmesi.

Bu üç parça bir araya geldiğinde Zabbix’in VMware template’i  eksiksiz ve en az yetki prensibiyle çalışır.

## Kaynaklar

**Groups in the vCenter Single Sign-On Domain** `https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-sdks-tools/8-0/groups-in-the-vsphere-local-domain.html`

**Add a User to the SystemConfiguration.BashShellAdministrators Group** `https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere/8-0/vcenter-configuration/configuring-vcenter-server-using-the-vsphere-client/add-a-user-to-the-systemconfiguration-bashshelladministrators-group.html`

**Using vCenter Server Global Permissions** `https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere/8-0/vsphere-security/vsphere-permissions-and-user-management-tasks/global-permissions.html`

**Understanding Authorization in vSphere** `https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere/8-0/vsphere-security/vsphere-permissions-and-user-management-tasks/understanding-authorization-in-vsphere.html`

**Zabbix VMware entegrasyon dokümantasyonu** `https://www.zabbix.com/integrations/vmware`

