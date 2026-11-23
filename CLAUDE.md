# Fork do OneFamily: vários controles USB no PS4

Contexto passado de uma sessão anterior (análise feita em 23/11/2026). Leia antes de mexer no código.

## Objetivo

Fazer **2 controles USB/dongle funcionarem ao mesmo tempo** no daemon do OneFamily (PS4 com GoldHEN):

- **Flydigi Vader 3 Pro**: dongle 2.4 GHz em modo XInput
- **GameSir Cyclone 2**: dongle 2.4 GHz em modo XInput (perfil existente: `profiles/3537_100B.ini`)

Os dois caem no caminho **Xbox 360 / XInput** (`CONTROLLER_XBOX360`, detectado pela subclasse de interface 0x5D em `controller_detect.c`).
O GameSir G7 está **fora do escopo** por enquanto, porque não funciona nem sozinho.
Bluetooth também está fora do escopo: o driver BT só serve para DualSense e depende de offsets de kernel por firmware.

Divisão do trabalho: Claude edita e **compila** localmente. A usuária testa no console e devolve o `/data/onefamily/onefamily.log`.

## Base do código

- Upstream: https://github.com/ChiricoG/OneFamily (licença MIT)
- Usar o branch **`dev`** (v1.2.4-beta1, commit `ba0975b`, 14/11/2026), mais novo que o `main` (v1.2.3).
- O trabalho fica num **fork público** no GitHub (`origin`), com o repositório original como remote `upstream`. Desenvolver num branch próprio (`multi-usb`) criado a partir do `dev` e rebasear sobre `upstream/dev` quando o autor publicar novidades. Como o fork é público, a mudança pode virar um PR para o upstream depois (é a meta da v1.3.0).

## Diagnóstico (issue #74)

A camada de saída **já aceita vários controles**. O gargalo é só `daemon/src/usb_driver.c` (~2.350 linhas).

Já pronto:
- `vpad.c`: 4 slots (`VPAD_MAX_SLOTS`), vibração por slot, vínculo de usuário por slot, `vpad_slot_open_dev(source, device_key, &slot)`.
- `bt_driver.c`: já usa `s_devices[BT_MAX_DEVICES]`. Serve de modelo para a estrutura por dispositivo.
- `daemon_syscalls.c`: `daemon_kernel_thread_create()` (scePthreadCreate) já resolvido.

Problemas em `usb_driver.c`:
1. Um único handle `static libusb_device_handle *g_dev_handle` (linha ~27) e a struct global `g_usb_state` (`usb_driver.h`).
2. `scan_and_open()` (~linha 1087) faz `if (g_dev_handle != NULL) return 0;` logo no início, então o 2º dongle nunca é examinado.
3. Cerca de 40 variáveis `static` com estado de um dispositivo só. Para o caminho 360 importam: `s_consecutive_*`, `s_x360_ep_out_stalls`, `s_x360_wireless_present`, `s_last_ds4`, `s_usb_vpad_slot`, `s_share_combo_active`, `s_last_guide_pressed`, `s_logged_*`, regras de remapeamento (`s_remap_rules`), `s_usb_connect_time_us`, `g_claimed_interface`, `g_cur_dev`.
4. O slot é aberto com `vpad_slot_open(VPAD_SOURCE_USB, &s_usb_vpad_slot)` (~linha 1429), sem chave de dispositivo.
5. `is_current_device_present_on_bus()` (~linha 1680) compara **VID/PID**. Os dois dongles podem enumerar como `045E:028E`, então **identificar por porta/endereço, não por VID/PID**.
6. `usb_driver_poll_step()` (~linha 1849) faz leitura bloqueante com timeout de 8 a 16 ms. Controles 360 parados só reportam quando algo muda, então ler 2 ou mais em série soma os timeouts.
7. `usb_driver_send_rumble()` e o callback de vibração do vpad mandam para o único dispositivo.
8. `main.c` usa `g_usb_state` para o reload de perfil HID e para o ritmo do loop (1000 Hz ou 250 Hz).
9. `generic_hid.c` tem contexto único (`s_ctx`, `s_state`). Só importa se algum controle for usado em modo D-input, o que não é o plano.

## Plano

0. **Ambiente de build:** `build_daemon.ps1` e `build_app.ps1` têm caminhos fixos da máquina do autor (`$TOOLCHAIN`, `$PYTHON`, `$CREATE_FSELF = C:/tmp/create-fself-new.exe`). Parametrizar por variável de ambiente. O `create-fself` vem do OpenOrbis (`bin/windows`). Pré-requisitos: OpenOrbis PS4 Toolchain, LLVM clang/ld.lld, .NET 8/9 SDK (PkgTool.Core), Python 3. Primeiro objetivo: compilar o `dev` sem nenhuma alteração.
1. **Teste de base:** cada controle sozinho no beta1 e o log guardado.
2. **Estado por dispositivo:** criar `typedef struct UsbPad { ... } UsbPad;` com tudo do item 3 acima, mais handle, `g_usb_state` e o slot do vpad, num array `s_pads[USB_MAX_PADS]` (2, depois 4). Para mexer pouco no código, usar um ponteiro `s_cur` mais macros (`#define g_dev_handle (s_cur->dev_handle)` etc.) e trocar `s_cur` antes de processar cada pad. Cobrir só o caminho XBOX360. Os outros tipos continuam limitados ao pad 0.
3. **Varredura:** `scan_and_open()` deixa de sair cedo. Para cada dispositivo ainda não reivindicado, pega um `UsbPad` livre. Chave do dispositivo: ponteiro `libusb_device*` ou barramento+endereço (verificar se `libSceUsbd` exporta `sceUsbdGetBusNumber`/`sceUsbdGetDeviceAddress` com `daemon_sys_dlsym`). Abrir o slot com `vpad_slot_open_dev(VPAD_SOURCE_USB, key, &pad->vpad_slot)`.
4. **Polling:** v1 em rodízio com timeout curto (1 a 2 ms; o round-trip do sceUsbd fica entre 2 e 3,5 ms, segundo um comentário no código). v2 com uma thread por pad (`daemon_kernel_thread_create`) e mutex em volta de `vpad_send_data_slot`.
5. **Vibração:** levar a vibração por slot do `vpad_poll_rumble()` até o `UsbPad` dono do slot, como já é feito com `bt_driver_send_rumble_slot`.
6. **Desconexão e hotplug:** fechar só o pad que sumiu e manter o outro.
7. **Hardware:** o PS4 tem 2 portas USB (o Pro tem 3). Hubs (`bDeviceClass 0x09`) já são ignorados na varredura, e os dispositivos atrás deles são lidos.

## Ciclo de teste

- Para testar só o daemon, não precisa reinstalar o PKG: basta substituir `/data/payloads/onefamily.bin` no console por FTP e reiniciar (ou recarregar o payload).
- Log: `/data/onefamily/onefamily.log`. Adicionar logs com o índice do pad em cada evento (claim, slot, desconexão).
- Pendências com a usuária: modelo e firmware do PS4, e o VID:PID de cada dongle no PC (USBTreeView).
