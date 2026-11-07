# Ricerca web/GitHub — chi ha messo mano al C60 (e ai cugini Telink)

> Ricerca del 2026-11-07. Ogni voce ha il link. **[CONFERMATO]** = pagina/codice letti;
> **[NON VERIFICATO]** = solo snippet di ricerca o affermazione del venditore.

## Esito in una riga

**Nessuno ha pubblicato teardown, dump, firmware custom o lavori OTA sul C60 (LIGE o altri
marchi).** L'unico progetto che tocca proprio questo dispositivo è **Gadgetbridge**, e solo lato
app (protocollo). Le stringhe `TB2C60T7`, `TC2C60T7`, `C60-A82C` non danno risultati. Sul
firmware siamo i primi.

## A. Specifici C60 / KeepHealth

| # | Fonte | Cosa | Stato |
|---|---|---|---|
| 1 | [Gadgetbridge – Keep Health](https://gadgetbridge.org/gadgets/wearables/keephealth/) | LIGE C60 "Mostly supported": ora, profilo, impostazioni, notifiche/chiamate, fotocamera, trova; sync passi, HR, SpO2, sonno, temperatura, pressione. Niente sveglie e meteo | CONFERMATO |
| 2 | [Gadgetbridge PR #5634](https://codeberg.org/Freeyourgadget/Gadgetbridge/pulls/5634) (LiJu09, merge 2026-05-29, `637ca0069c`) | `C60Constants.java`, `C60DeviceSupport.java` (1533 righe), `C60DeviceCoordinator.java`. Conferma FF00/FF02/FF01, checksum `(sum*86)+90` = il nostro `(sum·0x56)+0x5A`, frame cmd + len 16 bit + dati + chk, nome `^C60-.*`. Nessun firmware/OTA | CONFERMATO |
| 3 | Manuali/schede: [manuals.plus 1005010016657103](https://manuals.plus/ae/1005010016657103), [primini.tn](https://primini.tn/prix/177988/bracelet-connectee-lige-c60-noir), [tvc-mall](https://www.tvc-mall.com/details/c60-1-1-inch-waterproof-smart-watch-heart-rate-blood-oxygen-monitor-body-temperature-detection-fitness-tracker-sports-smart-wristband-grey-sku680501242c.html) | 1,1" **AMOLED** 126×294, 90 mAh, BLE 5.0, IP67. "C60" venduto anche da EFFEOKKI, Aukire, BSLIZHEN, ST-CARE, Cubonic (hardware non per forza uguale) | CONFERMATO (specifiche) |
| 4 | [Alibaba "New C60 Smart Band 1.1 Inch Amoled"](https://www.alibaba.com/product-detail/New-C60-Smart-Band-1-1_1600743351365.html) | lo snippet dice chipset **Telink TLSR8251F512ET32** (coincide con le nostre note) | NON VERIFICATO |
| 5 | [KeepHealth Pro (App Store)](https://apps.apple.com/ae/app/keephealth-pro/id6443824485) | solo contesto | — |

Due cose utili da Gadgetbridge:

- nei commenti, gli UUID copiati dall'app: OTA **service `…1912`**, char `…2b12`; servizio versione
  `0000d0ff-3c17-d293-8e48-14fe2e4da212` / `ffd4`. Anche `C60-A82C.md` in c60emu riporta il
  servizio `…1912` letto dal device, mentre `run-native-firmware-update.md` scrive `…1911`:
  **va controllato quale UUID di servizio usa davvero `ota_upload.py`**;
- lingue: en 0, zh 1, ru 2, fr 4, es 5, de 7, ja 8, pl 9, **it 10**, zh_TW 12, nl 15.

Display AMOLED: il pannello parla un set comandi tipo ST7789 ma il driver è probabilmente un IC
AMOLED (deduzione, non confermata).

## B. Orologi/bande Telink economici smontati o riprogrammati

| # | Progetto | Chip | Risultato |
|---|---|---|---|
| 6 | [rbaron/m6-reveng](https://github.com/rbaron/m6-reveng) ([articolo](https://rbaron-net.pages.dev/blog/2021/07/06/Reverse-engineering-the-M6-smart-fitness-band), [Hackaday](https://hackaday.com/2021/07/08/reverse-engineering-a-very-cheap-fitness-band/)) | TLSR8232 | debugger SWire in Python + Blue Pill, dump 512 KB, toolchain tc32 in Docker |
| 7 | [e-labInnovations/FitPro-LT715-TLSR8232](https://github.com/e-labInnovations/FitPro-LT715-TLSR8232) ([blog](https://elabins.com/blog/hacking-a-249-smartwatch-custom-firmware-on-tlsr8232)) | TLSR8232 | SWire, driver, firmware originale decompilato in C ricompilabile (Ghidra), firmware OEPL. Base di lt716 |
| 8 | [davidgiven/telinkdebugger](https://github.com/davidgiven/telinkdebugger) | TLSR8232 (LT716) | ponte SWS su Raspberry Pi Pico, lettura/scrittura flash |
| 9 | MicroPython su un orologio da 3 $ ([Hackaday](https://hackaday.com/2024/06/26/3-smartwatch-can-run-python/), [Hackster](https://hackster.io/news/coding-time-aac8b15f6d64), [Habr](https://habr.com/en/posts/824264)) | TLSR8232 (LT716) | teardown, dump, port MicroPython |
| 10 | [HackSucks/OpenWearOTA](https://github.com/HackSucks/OpenWearOTA) ([PyPI](https://pypi.org/project/openwearota/), [OpenWearEMU](https://github.com/HackSucks/OpenWearEMU)) | Telink 8232/8253/82xx | **firmware custom via OTA del produttore, senza aprire**. Stessi UUID `…1912`/`…2b12`/`d0ff`/`ffd4` del KeepHealth. Avverte: se il firmware custom non reimplementa l'OTA, l'OTA è perso |
| 11 | atc1441: [ATC_MiThermometer](https://github.com/atc1441/ATC_MiThermometer), [ATC_TLSR_Paper](https://github.com/atc1441/ATC_TLSR_Paper), [Xiaomi-TLSR-Firmwares](https://github.com/atc1441/Xiaomi-TLSR-Firmwares), [smart-watch-socs](https://github.com/atc1441/smart-watch-socs), [Gadgetbridge-BLE-OpenEpaperLink](https://github.com/atc1441/Gadgetbridge-BLE-OpenEpaperLink) | TLSR825x vari | OTA web, firmware custom; nessun orologio C60 |

## C. Strumenti Telink

- **[pvvx/TLSR825x_OTA_Flash_Hacker](https://github.com/pvvx/TLSR825x_OTA_Flash_Hacker)**
  ([UI web](https://pvvx.github.io/CGPR1/TLSR825xOTA_Hacker.html)) — **il precedente più vicino al
  nostro dumper**. Si flasha `BLE-Hacker.bin` con l'OTA Telink standard, poi legge tutta la flash,
  scrive, cancella settori, pulisce l'area OTA e con "Change Start Boot" rimette il firmware
  originale. Nel codice (`cmd_parser.c`): cerca `KNLT` a +8 di `0x0…0x40000`; per cambiare banco
  scrive `KNLT` a +8 dell'altro e 0 a +8 del corrente. Assume 512 KB e multi-boot ROM: **sul C60
  (boot del produttore a `0x0`, app a `0x14000`, appoggio esterno a `0x374000`) non si usa così
  com'è**.
- SWire: [TlsrComSwireWriter](https://github.com/pvvx/TlsrComSwireWriter) (solo USB-UART),
  [TLSRPGM](https://github.com/pvvx/TLSRPGM) ([issue dump TLSR8251](https://github.com/pvvx/TLSRPGM/issues/2)),
  [TlsrTools825x](https://github.com/pvvx/TlsrTools825x), [TlsrComProg825x](https://github.com/pvvx/TlsrComProg825x),
  [TlsrTools](https://github.com/pvvx/TlsrTools) (protocollo SWire).
- OTA web/CLI: [TelinkOTA pvvx](https://pvvx.github.io/ATC_MiThermometer/TelinkOTA.html),
  [tlsr825x-telink-ota-fast-logger](https://github.com/Cardistalive/tlsr825x-telink-ota-fast-logger),
  [TelinkOTAFlasher](https://github.com/EnderDragonEP/TelinkOTAFlasher),
  [telink_burn_tool](https://github.com/FilipeCarvalhedo/telink_burn_tool) (BDT reversato),
  [telink-usb-ota](https://github.com/jbonadiman/telink-usb-ota).
- TC32: Ghidra [rgov](https://github.com/rgov/Ghidra_TELink_TC32), [trust1995](https://github.com/trust1995/Ghidra_TELink_TC32),
  [leommxj](https://github.com/leommxj/ghidra_tc32); [modern-tc32/tc32-architecture](https://github.com/modern-tc32/tc32-architecture),
  [open-tc32-gcc-4.5](https://github.com/modern-tc32/open-tc32-gcc-4.5); [goyamamoto/tc32-devtools](https://github.com/goyamamoto/tc32-devtools)
  (clang + emulatore: da confrontare con c60emu); [TelinkDev/toolchain_binary](https://github.com/TelinkDev/toolchain_binary),
  [flyskywhy/tc32](https://github.com/flyskywhy/tc32); [openocd_tc32](https://github.com/MD2020nMe/openocd_tc32).
- SDK B85: [SoCXin/TLSR8258](https://github.com/SoCXin/TLSR8258), [mztulip/TLSR8251_example](https://github.com/mztulip/TLSR8251_example),
  [Ai-Thinker-Open/Telink_825X_SDK](https://github.com/Ai-Thinker-Open/Telink_825X_SDK).

## D. Avvio e OTA della B85 (sorgenti ufficiali Telink)

- [`tc_ble_mesh` B85 `mcu_boot.h`](https://github.com/telink-semi/tc_ble_mesh/blob/655644f04f9200c6fd55cc456f874fa2042c9ab6/firmware/drivers/B85/driver_ext/mcu_boot.h):
  multi-boot solo `0x20000` e `0x40000`; `BOOT_MARK_VALUE 0x4B` a `0x08`; dimensione a `0x18`.
- [`ext_ble_common.c`](https://github.com/telink-semi/tc_ble_mesh/blob/master/firmware/stack/ble/ble_ext/ext_ble_common.c),
  `ota_set_flag()`: scrive `0x4B` a +8 della nuova immagine (che arriva con quel byte non
  impostato), azzera +8 del banco vecchio — **tranne** con `PINGPONG_OTA_DISABLE` o
  `FW_START_BY_LEGACY_BOOTLOADER_EN`, dove il vecchio resta. Banco avviato:
  `((REG_ADDR8(0x63e)&3)<<7)<<10`. Protezione: scrive solo fuori dall'immagine in esecuzione.
- Nello stesso repo, `boot.link` B85 ha le opzioni **`__BOOT_LOADER_EN`** e
  **`__FW_START_BY_LEGACY_BOOTLOADER_EN`**: l'SDK prevede un bootloader in flash con l'app a un
  offset diverso da 0. **Probabilmente è lo schema del C60** (boot `0x0–0x14000`, app `0x14000`),
  ma è un'ipotesi.
- [`telink_zigbee_sdk` 8258 `ota.h`](https://github.com/telink-semi/telink_zigbee_sdk): opcode
  `0xFF00` versione, `0xFF01` start, `0xFF02` end, `0xFF03` start_ext, `0xFF04/05` versione,
  `0xFF06` risultato. Coincidono con gli indici gestiti da `ota_data_write` (`0xff00/01/03/04`).
- [B85m BLE Single Connection SDK Handbook](https://wiki.telink-semi.cn/doc/an/AN-21112300-E_Telink%20B85m%20BLE%20Single%20Connection%20SDK%20Developer%20Handbook.pdf),
  §7.1.2 (flag OTA, OTA_END): **non letto**, il certificato del sito è scaduto. Da scaricare a mano.

## E. Altri ecosistemi (nessun legame col protocollo C60)

[lcd-badge-ble](https://github.com/DynamicDevices/lcd-badge-ble), [superband-web-client-rnd](https://github.com/T-REX-XP/superband-web-client-rnd),
[openwatch-LY736D](https://github.com/mehranshahmiri/openwatch-LY736D), [ebadge](https://github.com/samascience/ebadge) (JieLi),
[FitPro M4/M5/M7 con Frida](https://mandomat.github.io/2026-02-04-speaking-the-language-of-wearables).
Nessuno usa i frame FF01/FF02 con checksum `·0x56+0x5A`. Nessun protocollo Da Fit, FitCloudPro,
GloryFit, H Band o Wearfit corrisponde.

## Non coperto

4PDA e elektroda non verificati ([questa discussione](https://www.elektroda.pl/rtvforum/topic4153250.html)
andava in timeout); nessun thread rilevante su Reddit/XDA. SoC, flash esterna, driver display e
sensore HR di un C60 restano senza foto pubbliche.
