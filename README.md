# DLSS sob estresse prolongado: Windows nativo vs Linux + Proton

Dados brutos de um experimento de **envelhecimento de software** (*software aging*): uma cena do Unreal Engine 5 com **NVIDIA DLSS** ativo rodou sem parar por **8 horas** em duas plataformas, na mesma máquina, enquanto um script em Python registrava métricas de desempenho e de consumo de recursos a cada 5 segundos.

A pergunta do experimento é se o desempenho e o uso de recursos (FPS, memória, VRAM, temperatura) se degradam ao longo de uma execução longa, e se esse comportamento muda entre o **Windows 10 nativo** e o **Linux rodando o jogo via Proton** (camada de compatibilidade que traduz DirectX 12 para Vulkan).

Os dados fazem parte da pesquisa *Software-Aging-Benchmark* e foram publicados aqui para o projeto final da disciplina **Computação para Análise de Dados** (UFRPE, 2026.2).

- **Autor:** João Lucas
- **Coleta:** 31/10/2026 (Linux) e 02/11/2026 (Windows)
- **Licença:** CC BY 4.0 (veja `LICENSE`)

---

## Estrutura do repositório

```
2026-10-31_12-31_linux/
├── 02_dlss_linux.csv                    # métricas a cada 5 s (base principal)
├── 02_dlss_processos_linux.csv          # processos mais ativos do sistema, a cada ~60 s
├── 02_dlss_eventos_sistema_linux.txt    # avisos/erros do journal do sistema durante o teste
├── log_linux.txt                        # log do script que conduziu o teste
├── sessao.json                          # metadados: máquina, versões, configuração do teste
└── relatorio/                           # resumos e gráficos gerados pelo script (já processados)

2026-11-02_14-50_windows/
├── 02_dlss_windows.csv
├── 02_dlss_processos_windows.csv
├── 02_dlss_eventos_sistema_windows.txt
├── log_windows.txt
├── sessao_windows.json
├── logs/                                # logs da Unreal (um por execução do jogo)
└── relatorio/
```

O nome de cada pasta segue o padrão `AAAA-MM-DD_HH-MM_<plataforma>`, com a data e a hora local (UTC−3) do início do teste.

---

## Condições do experimento

| Item | Valor |
|---|---|
| CPU | Intel Core i7-7700K @ 4.20 GHz |
| GPU | NVIDIA GeForce RTX 3070 Ti |
| RAM | 16 GB |
| Windows | Windows 10 (build 19045), driver NVIDIA 595.97 |
| Linux | Kernel 7.0.0-42-generic (glibc 2.39), driver NVIDIA 595.99.02 |
| Camada de compatibilidade (Linux) | GE-Proton11-11, iniciado com `umu-run` |
| Motor | Unreal Engine 5.4.4 |
| Plugins | DLSS 8.6.1 (NGX 310.6.0), Streamline 8.6.1 (SL 2.11.1) |
| Cena | build do projeto UE5-Benchmark (câmera em trajeto fixo, em loop) |
| Resolução | 2560×1080, tela cheia |
| API gráfica | DirectX 12 (no Linux, traduzida para Vulkan pelo Proton) |
| Ray tracing por hardware | ativado |
| DLSS | Super Resolution no modo **Quality**; Frame Generation **desligado** (não suportado pela RTX 3070 Ti) |
| VSync / limite de FPS | desligado / sem limite |
| Atualização automática (OTA) do DLSS | desativada, para garantir a mesma versão nas duas plataformas |
| Duração | 8 h por plataforma |
| Intervalo de amostragem | 5 s (base principal); ~60 s (processos) |

Todos os valores acima vêm dos arquivos `sessao*.json`.

---

## Dicionário de dados

### Base principal: `02_dlss_<plataforma>.csv`

Uma linha por amostra (a cada ~5 s). Separador: vírgula. Decimal: ponto. Valores ausentes: campo vazio.

**Identificação da amostra**

| Coluna | Tipo | Unidade | Descrição |
|---|---|---|---|
| `timestamp_utc` | data/hora | ISO 8601, UTC | Data e hora da amostra, no fuso UTC (3 h a mais que o horário de Brasília/Recife), com milissegundos (ex.: `2026-10-31T15:31:54.918Z`) |
| `horas` | numérico | h | Horas passadas desde o início do teste |
| `execucao` | inteiro | – | Número da execução do jogo. Começa em 1 e só aumenta se o jogo travar e for reaberto |

**Desempenho** (medido pelo jogo, média dos últimos 5 s)

| Coluna | Tipo | Unidade | Descrição |
|---|---|---|---|
| `fps` | numérico | quadros/s | Quadros por segundo renderizados pela GPU |
| `fps_exibido` | numérico | quadros/s | Quadros por segundo mostrados na tela com Frame Generation. Fica vazio quando o Frame Generation está desligado, como neste teste |
| `frametime_p99_ms` | numérico | ms | Tempo de frame no percentil 99. Mostra as travadas: 1% dos frames demorou isso ou mais |
| `gpu_time_ms` | numérico | ms | Quanto tempo a GPU levou para desenhar cada frame |
| `game_thread_ms` | numérico | ms | Tempo da thread de lógica do jogo (CPU) por frame |
| `render_thread_ms` | numérico | ms | Tempo da thread que prepara os comandos de desenho (CPU) por frame. Se for maior que `gpu_time_ms`, o limite é a CPU |

**Processo do jogo** (medido pelo sistema operacional)

| Coluna | Tipo | Unidade | Descrição |
|---|---|---|---|
| `proc_mem_mb` | numérico | MB | Memória física usada pelo jogo (Working Set no Windows, RSS no Linux) |
| `proc_private_mb` | numérico | MB | Memória alocada só pelo jogo (Private Bytes). É o principal indicador de vazamento de memória |
| `proc_threads` | inteiro | – | Número de threads do jogo |
| `proc_handles` | inteiro | – | Recursos do sistema abertos pelo jogo. No Windows são handles; no Linux, descritores de arquivo. Se sobe sem parar, há vazamento de recursos |
| `proc_cpu_pct` | numérico | % | Uso de CPU do jogo, em % da CPU inteira (100 = todos os núcleos) |

**CPU da máquina**

| Coluna | Tipo | Unidade | Descrição |
|---|---|---|---|
| `cpu_total_pct` | numérico | % | Uso total da CPU, média de todos os núcleos |
| `cpu_nucleo_max_pct` | numérico | % | Uso do núcleo mais ocupado. Perto de 100% com o total baixo significa gargalo numa thread só |
| `outros_cpu_pct` | numérico | % | CPU usada por todos os outros processos juntos (total menos o jogo) |

**Memória do sistema e da GPU**

| Coluna | Tipo | Unidade | Descrição |
|---|---|---|---|
| `ram_mb` | numérico | MB | RAM usada no computador inteiro |
| `vram_mb` | numérico | MB | Memória de vídeo usada na GPU |

**GPU** (contexto)

| Coluna | Tipo | Unidade | Descrição |
|---|---|---|---|
| `gpu_util_pct` | numérico | % | Uso da GPU. Perto de 100% significa GPU no limite |
| `gpu_temp_c` | numérico | °C | Temperatura da GPU |
| `gpu_clock_mhz` | numérico | MHz | Frequência de trabalho da GPU |
| `gpu_power_w` | numérico | W | Potência consumida pela GPU |
| `throttle_power` | 0/1 | – | 1 = a GPU estava reduzindo o clock por limite de potência; 0 = não |
| `throttle_thermal` | 0/1 | – | 1 = a GPU estava reduzindo o clock por temperatura; 0 = não |

### Processos: `02_dlss_processos_<plataforma>.csv`

Processos do sistema que mais consumiam recursos, registrados a cada ~60 s. Uma linha por processo por leitura.

| Coluna | Tipo | Unidade | Descrição |
|---|---|---|---|
| `timestamp_utc` | data/hora | ISO 8601, UTC | Momento da leitura, **sem milissegundos e com offset** (ex.: `2026-10-31T15:31:31+00:00`) |
| `horas` | numérico | h | Tempo decorrido desde o início do teste |
| `pid` | inteiro | – | Identificador do processo no sistema operacional |
| `processo` | texto | – | Nome do processo (no Windows, com extensão `.exe`) |
| `cpu_pct` | numérico | % | Uso de CPU do processo |
| `mem_mb` | numérico | MB | Memória usada pelo processo |

### Metadados: `sessao*.json`

Descrevem a sessão: plataforma, início e fim, máquina (`maquina`), versões de software (`versoes`), configuração (`configuracao`) e, para cada teste (`testes`), as *cvars* da Unreal aplicadas, o modo do DLSS, o número de execuções e reinícios, a quantidade de amostras e os problemas encontrados.

---

## Peculiaridades conhecidas

- **`fps_exibido` está 100% vazia** nas duas bases, porque o Frame Generation estava desligado.
- **A primeira amostra de cada execução** tem `proc_cpu_pct` e `outros_cpu_pct` vazios e `cpu_total_pct` = 0: o uso de CPU é calculado pela diferença entre duas leituras e ainda não havia leitura anterior.
- **Os primeiros minutos são de aquecimento**: o FPS, a memória e a temperatura ainda estão subindo enquanto a cena carrega e a GPU esquenta.
- **No Windows, o jogo fechou sozinho uma vez**, após ~3 minutos (código de saída 0), e o script o reabriu. Por isso, nessa base, `execucao` assume os valores 1 e 2, e há um pequeno intervalo sem amostras. O Linux rodou as 8 h sem reinício.
- **Os timestamps estão em UTC.** O horário local de Recife é UTC−3.
- **O formato do timestamp difere entre os arquivos**: a base principal usa milissegundos e o sufixo `Z`; a de processos usa segundos e o sufixo `+00:00`.
- **Os nomes de processo não seguem o mesmo padrão** entre os sistemas (ex.: `MsMpEng.exe` no Windows, `gnome-shell` no Linux).
- **A métrica `proc_handles` não é diretamente comparável** entre as plataformas: no Windows conta handles do sistema; no Linux, descritores de arquivo do processo rodando sob o Proton.
- **O executável é o mesmo nas duas plataformas** (build Windows da cena). No Linux ele roda pelo Proton.
- **O arquivo de eventos do Windows** foi exportado com uma codificação de caracteres diferente de UTF-8, e alguns acentos aparecem corrompidos (ex.: `Informaþ§es`).

---

## Como citar

> João Lucas. *DLSS sob estresse prolongado: Windows nativo vs Linux + Proton* [conjunto de dados]. GitHub, 2026. Disponível em: https://github.com/Jlucas66/Projeto_2VA_Dados