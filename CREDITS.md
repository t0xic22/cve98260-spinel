# Créditos

Este repo contém o exploit **CVE-2026-98260** adaptado ao Redmi Note 15 4G (spinel).
O código original pertence aos autores abaixo; aqui há apenas o perfil do spinel
e modificações experimentais locais.

## Autores originais

- **[YuKongA/CVE-2026-98260](https://github.com/YuKongA/CVE-2026-98260)** — repo
  original, timer LPE + perfis por `uname -r`. **MIT** (`LICENSE`).
- **[woshigenm/CVE-2026-98260](https://github.com/woshigenm/CVE-2026-98260)** —
  `src/uid0.c` e `src/poc.c` (MIT), base declarada do `timer.c` no README upstream.
- **`tools/extract/`** —originalmente de
  **[YuKongA/ghostlock-app](https://github.com/YuKongA/ghostlock-app)** (`Apache-2.0`).

## Modificações locais

- `timer/kbuilds/6.12.69-android16-6-g014ca48a80ee-ab15889839-4k.h` — perfil do
  spinel, derivado de análise do boot.img e do BTF do device
  (`root-spinel-notes/fw/btf.bin`, extraído com `parse_btf.py`).
- `timer/timer.c` — handoff de KernelSU, script de props, modo `perm`, instrumentação
  de capabilities e variantes v2–v25.
- As licenças originais (MIT / Apache-2.0) permanecem nos arquivos `LICENSE` e `NOTICE`.

## Agradecimentos

- **[RikkaApps/Shizuku](https://github.com/RikkaApps/Shizuku)** e
  **[thedjchi/Shizuku](https://github.com/thedjchi/Shizuku)** — uid 2000 para o app.
- Comunidade HyperOS — documentação de `setprop` via MQS e `task_stack_view_layout_style`.

Veja também os créditos em
[`ghostlock-spinel`](https://github.com/t0xic22/ghostlock-spinel/blob/main/CREDITS.md)
e [`root-spinel-notes`](https://github.com/t0xic22/root-spinel-notes/blob/main/CREDITS.md).
