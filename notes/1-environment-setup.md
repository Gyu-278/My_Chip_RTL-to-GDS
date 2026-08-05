# 환경 Setting

- 대상: https://github.com/GoodKook/ETRI-0.5um-CMOS-MPW-Std-Cell-DK
- 강좌: ETRI 국일호 책임연구원 3일 집중 강좌
- 서버: `turtle` (Rocky Linux 8.10)

### Step 0. 목표

- **최종 목표**: Verilog HDL로 디지털 회로를 기술한 후, SystemC(C++)로 시스템 수준 검증을 수행한다. 이후, 합성/APR을 거쳐 **MPW에 제출 가능한 GDS 생성**까지 디지털 반도체 설계 전 과정을 오픈소스만으로 완주하는 것을 목표로 한다.
- **예제**: 강좌의 **"탁구 게임기(Pong)"** : RTL → SystemC 테스트벤치(그래픽 LCD를 C++로 모델링, 비트맵으로 결과 시현) → 합성/타이밍 시뮬(VPI) → APR/GDS → Chip-Top.
- **방향성**: SkyWater 130nm 오픈PDK의 "국내 ETRI 0.5um 버전". 상용 EDA(Cadence 등) 없이 100% 오픈소스로 실제 태이프아웃이 가능. 개인이 저렴하게 MPW로 실물 칩을 찍어볼 수 있음.
- 킷이 요구하는 오픈소스 EDA 툴 체인
    
    ```
    RTL → 합성(Yosys) → 배치(GrayWolf) → 라우팅(QRouter)→ 레이아웃(Magic) → LVS 검증(Netgen) → 시뮬(ngSpice)
    전체를 QFlow가 묶고, 검증에 iverilog/verilator/gtkwave, 시스템검증에 SystemC 사용.
    ```
    

### Step 1. 서버 환경 점검

- 본 서버에서 킷을 돌릴 수 있는지(OS·권한·네트워크·빌드툴·자원)부터 확인.
    
    ```bash
    cat /etc/os-release | head -2      # OS
    whoami ; sudo -n true              # 권한
    curl -sI --max-time 8 https://github.com | head -1   # egress
    nproc ; free -h | head -2          # 자원
    for t in git gcc g++ make cmake python3 tcl ; do command -v $t ; done  # 빌드툴
    ```
    
- 결과:
    
    ```
    NAME="Rocky Linux"  VERSION="8.10 (Green Obsidian)"
    whoami: pgh   |   sudo: a password is required   # ← sudo 없음
    HTTP/2 200                                       # ← github egress 정상
    48 코어 / Mem 251Gi (available 238Gi)
    git /usr/bin/git | gcc 8.5.0 | g++ | make | cmake | python3 /hai/anaconda3 | tcl: 없음
    ```
    
- **Step 1 결론**: 자원·네트워크·빌드툴은 충분하나, **(1) 내 계정에는 sudo 없음, (2) OS가 데비안 계열이 아니기에** 기존의 강좌 공식 설치법을 그대로 못 씀 (Step 2).

### Step 2. 문제 확인.

- 강좌 사전준비 가이드([WSL 설치 안내](https://fun-teaching-goodkook.blogspot.com/2025/12/20-wsl.html))와 킷의 `Tools/build_tools.sh`는 **Ubuntu(WSL)의 `apt` + `sudo` 전제**로 작성됨.
- 한편, 연구실 서버의 경우,
    1. **데비안 계열이 아니라 `apt`가 없음** (`dnf` 기반). → apt 명령 전부 실패.
    2. **sudo가 없음.** → 시스템 전역(`/usr/local` 등) 설치 불가.
    
    ```bash
    command -v apt-get   # → 없음 (데비안 계열 아님)
    command -v dnf       # → /usr/bin/dnf (있지만 sudo 필요 → 못 씀)
    ```
    
- **결정**: 일단, 내 **conda 환경 안에만** EDA 툴을 몰아넣어 활용.

### Step 3. 필요한 대상 설치

- conda로 새로 깔 대상과 이미 있는 것을 구분해 헛설치를 피한다.
    
    ```bash
    for t in yosys magic netgen qrouter graywolf qflow klayout xschem \
             ngspice iverilog vvp verilator gtkwave systemc ; do
      command -v $t >/dev/null && echo "$t: 있음" || echo "$t: 없음"
    done
    ```
    
- 결과:
    
    
    | 툴 | 상태 | 설치 계획 |
    | --- | --- | --- |
    | `verilator, gtkwave` | ✅ **이미 있음** | 그대로 사용 |
    | `yosys, magic, netgen, klayout, ngspice, iverilog, xschem` | ❌ 없음 | conda 채널(litex-hub 등)로 sudo 없이 설치 |
    | `systemc` | ❌ 없음 | conda-forge 또는 소스 빌드 |
    | **`graywolf, qrouter, qflow`** | ❌ 없음 | **conda에 잘 없음 → 소스 빌드 필요** |
- **핵심 리스크 식별**: 이 킷은 구세대 **qflow 플로우**(graywolf 배치 + qrouter 라우팅)를 활용함.
- 부족한 3종은 conda 패키지가 마땅치 않아 `gcc 8.5` + `tcl/tk`로 홈에 소스 빌드.

### Step 4. 루트 설정 및 설치

본 폴더(`~/project/PGH_Chip_Open_Source`)에서 실행. 

- 킷 소스·표준셀·`Tools/` 빌드 스크립트 확보 및  Clone.
    
    ```bash
    cd /hai/home/pgh/project/PGH_Chip_Open_Source
    git clone --depth 1 https://github.com/GoodKook/ETRI-0.5um-CMOS-MPW-Std-Cell-DK.git
    ```
    
- `Tools/`에 **각 툴의 개별 소스 빌드 스크립트가 존재하는 것을 확인함.**
- `graywolf_build.sh`, `qrouter_build.sh`, `qflow-1.4.100_etri050_build.sh` 를 확인할 수 있으며, `qflow-1.4.100_etri050.tar.gz`(ETRI 커스텀 패치 qflow),`magic/netgen/yosys/systemc_build.sh` 등을 확인 가능.
    - `build_tools.sh`의 내용 확인
        - **graywolf** 자동 배치 / **qrouter** 자동 라우팅 / **qflow** 이 둘을 묶는 플로우
        - **magic** 레이아웃 편집 / **klayout** 레이아웃 뷰어 / **netgen** LVS 비교
        - **yosys** RTL 합성 / **OpenSTA** 정적 타이밍 분석
        - **ngspice** SPICE 시뮬 / **xschem** 스키매틱 입력 / **irsim** 스위치레벨 시뮬
        - **iverilog** Verilog 시뮬 / **verilator** Verilog→C++/SystemC 변환 / **systemc** 시스템 모델링 / **gtkwave** 파형 뷰어
    - 주요 과정 3종을 apt 없이 소스 빌드하도록 이미 준비된 것을 확인.
- 단, `Tools/prerequisites.sh`는 전부 `sudo apt install`(거대한 Ubuntu 의존성)이기에 필요한 것만 conda로 대체하여 활용.

### Step 5. conda env `chip-eda` 생성 + conda EDA 툴 설치·검증

- conda env(chip-eda) 생성
    
    ```bash
    conda create -n chip-eda -y python=3.11
    conda install -n chip-eda -y -c litex-hub -c conda-forge yosys magic netgen
    ```
    
- 결과 (실행 검증):
    
    ```
    yosys  0.65        ✅
    magic  8.3.465     ✅
    netgen 1.5.272     ✅
    ```
    

### Step 6. `graywolf` 소스 빌드

- 원본 `graywolf_build.sh`는 `clang` + `sudo make install` 전제로 되어있으며 해당 부분을 gcc + `CMAKE_INSTALL_PREFIX=$CONDA_PREFIX`로 변경.
- 빌드 중 3연속 경로 트러블슈팅 (conda prefix를 gcc/링커/pkg-config에 알려주는 과정):
    
    ```bash
    source scripts/activate-chip-eda.sh
    export PKG_CONFIG_PATH=$CONDA_PREFIX/lib/pkgconfig C_INCLUDE_PATH=$CONDA_PREFIX/include \
           LIBRARY_PATH=$CONDA_PREFIX/lib LD_LIBRARY_PATH=$CONDA_PREFIX/lib
    cd build/graywolf && mkdir build && cd build
    cmake -DCMAKE_C_COMPILER=/usr/bin/gcc -DCMAKE_CXX_COMPILER=/usr/bin/g++ \
          -DCMAKE_INSTALL_PREFIX=$CONDA_PREFIX ..
    make -j8 && make install      # sudo 없음
    ```
    
- 결과:
    
    ```
    [100%] Built target TimberWolfMC   # make 종료 0
    $ which graywolf → /hai/home/pgh/.conda/envs/chip-eda/bin/graywolf
    $ graywolf → "Available installed flow directories are:"  # 정상 usage 출력
    ```
    
- graywolf 설치 및 실행 완료. 빌드 의존성으로 conda에 `bison flex gsl tk` 추가 설치.
- `tcl`은 conda-forge 별도 패키지 없음 → `tk`가 tclsh/wish 포함.)

### Step 7. qrouter 소스 빌드

- graywolf와 동일 원칙. autotools라 `./configure`에 conda prefix + tcl/tk 경로 지정.

```bash
source scripts/activate-chip-eda.sh
export LD_LIBRARY_PATH=$CONDA_PREFIX/lib
cd build/qrouter
./configure --prefix=$CONDA_PREFIX --with-tcl=$CONDA_PREFIX/lib --with-tk=$CONDA_PREFIX/lib
make -j8 && make install
```

→ `qrouter` 설치·확인. (`no $DISPLAY` 경고는 GUI 모드용 정상 : qflow는 배치로 호출하므로 무관.)

### Step 8. qflow 빌드 + DK 심링크 + etri050 tech 연결

- 킷 전체가 `~/ETRI050_DesignKit/...` 경로(README의 "create symbolic link")를 가정하므로 **DK 심링크부터 생성**.

```bash
# ① DK 심링크 (킷의 모든 하드코딩 경로가 이걸로 해소됨)
ln -sfn <킷경로> ~/ETRI050_DesignKit
# ② qflow (동봉 커스텀 tarball) 빌드
cd build && tar xf <킷>/Tools/qflow-1.4.100_etri050.tar.gz && cd qflow-1.4.100_etri050
./configure --prefix=$CONDA_PREFIX && make && make install
```

- configure가 **"All tools needed for complete flow have been found"**
    
    (`yosys/graywolf/qrouter/magic/netgen` 전부 conda 것으로 검출). qflow v1.4.100 설치.
    
- **③ etri050 tech 연결** (빌드 스크립트 22~39행을 sudo 제거 + conda 경로로 개조)
    
    `$CONDA_PREFIX/share/qflow/tech/etri050`의 기본 tech를 `.bak`로 밀고, DK의 실제 파일 11개
    (`SCN3ME_SUBM.30.ETRI.tech`, `etri050_stdcells.lef`, `khu_etri05_stdcells.lib` 등)를 심링크 
    
    → 깨진 링크 0. **qflow가 실제 ETRI 0.5um PDK를 인식하도록 제작.**
    

### Step 9. 이외의 conda 툴 설치 완료 및 Back End Test 수행

```bash
conda install -n chip-eda -c litex-hub -c conda-forge klayout ngspice iverilog systemc gtkwave
conda install -n chip-eda -c conda-forge sdl2      # 탁구 게임 그래픽용
```

- 해당 과정에서 `klayout`,`ngspice`,`iverilog`,`systemc`,`gtkwave`,`SDL2` 설치 및 `verilator`는 시스템 것 사용.
- 이후, 툴이 개별로 설치됐어도 "합성이 실제 ETRI 셀로 매핑되는지"를 확인해야 함.
    - pong_pt1로 `config_m1f`(std-cell 구성) → `make synthesize` 시도.
    - Trouble Shooting
        - 합성 중 `share/techmap.v not found` 에러 발생. conda yosys는 `+/`(내장 데이터 경로)를 `$CONDA_PREFIX/share/`로 푸는데, 실제 데이터는`share/yosys/`에 있음(`YOSYS_DATDIR`도 무시).
        - 따라서, yosys 데이터 36개를 `share/`로 심링크. 하여 해결
        
        ```bash
        cd $CONDA_PREFIX/share && for i in yosys/*; do ln -s "yosys/$(basename $i)" "$(basename $i)"; done
        ```
        
- 결과: 재실행 시 pong_pt1이 ETRI050 표준셀 1004개로 합성 성공 (INVX1/NAND2X1/OAI21X1…),
ABC 매핑 → vlog2Verilog → vlog2Spice → SPICE 넷리스트까지 생성. `Synthesis script ended`.
- 정리: 스모크 테스트 잔여물(config_m1f의 셀 .mag 교체 + 합성 출력 68파일)은 `git checkout . && git clean -fd`로 킷을 pristine 복원하여 강좌를 탁구1부터 순서대로 할 수 있도록 제작.

### Step 10. 최종 툴 인벤토리 (env `chip-eda`)

| 백엔드(APR) | 시뮬/검증 | 레이아웃 |
| --- | --- | --- |
| `yosys`,`graywolf`,`qrouter`, `qflow(+etri050)`, `netgen` | `verilator`, `systemc`, `iverilog`, `gtkwave`, `ngspice`, `SDL2` | `magic`, `klayout` |