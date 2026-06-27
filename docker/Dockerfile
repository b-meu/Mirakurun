ARG ARCH=

# ── Stage 1: パッチ済み v4l-utils 1.32.0 のビルド ─────────────────────────────
#
# KouCo/v4l-utils-jp の変更内容を最新の v4l-utils 1.32.0 に適用する。
# （KouCo フォークは 1.22.1 ベースで古いため、パッチのみ移植する）
#
# 変更ファイル:
#   - dvb-v5-std.c : sys_dvbc_annex_b_props に DTV_SYMBOL_RATE を追加
#   - dvb-scan.c   : SYS_DVBC_ANNEX_B を NIT スキャン対象に追加
#                    ISDB-S NIT で DTV_STREAM_ID を設定
#   - dvb-fe.c     : SYS_DVBC_ANNEX_B のシンボルレート取得/設定を修正
#                    DVB-C/DVB-S2 で JP 国コード時に arib-std-b24 を設定
#
FROM ${ARCH}debian:bookworm-slim AS v4l-build

RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates git \
        meson ninja-build \
        gcc g++ \
        pkg-config gettext \
        libudev-dev \
        python3 \
        dpkg-dev \
    && rm -rf /var/lib/apt/lists/*

RUN git clone --depth 1 --branch v4l-utils-1.32.0 \
        https://github.com/gjasny/v4l-utils.git /src

WORKDIR /src

# JP パッチを Python スクリプトとして埋め込み、適用する
RUN cat > /tmp/patch.py << 'PYEOF'
def patch(path, old, new):
    with open(path) as f:
        c = f.read()
    if old not in c:
        raise ValueError(f"Pattern not found in {path}:\n{repr(old[:80])}")
    with open(path, 'w') as f:
        f.write(c.replace(old, new, 1))
    print(f"OK: {path}")

# ── dvb-v5-std.c: sys_dvbc_annex_b_props に DTV_SYMBOL_RATE を追加 ──────────
patch('lib/libdvbv5/dvb-v5-std.c',
    'const unsigned int sys_dvbc_annex_b_props[] = {\n'
    '\tDTV_FREQUENCY,\n'
    '\tDTV_MODULATION,\n'
    '\t0\n'
    '};',
    'const unsigned int sys_dvbc_annex_b_props[] = {\n'
    '\tDTV_FREQUENCY,\n'
    '\tDTV_MODULATION,\n'
    '\tDTV_SYMBOL_RATE,\n'
    '\t0\n'
    '};'
)

# ── dvb-scan.c: add_update_nit_isdbs に DTV_STREAM_ID を追加 ─────────────────
patch('lib/libdvbv5/dvb-scan.c',
    '\tdvb_store_entry_prop(new, DTV_SYMBOL_RATE,\n'
    '\t\t\t     d->symbol_rate);\n'
    '}\n'
    '\n'
    '\n'
    'static void __dvb_add_update_transponders',
    '\tdvb_store_entry_prop(new, DTV_SYMBOL_RATE,\n'
    '\t\t\t     d->symbol_rate);\n'
    '\tdvb_store_entry_prop(new, DTV_STREAM_ID,\n'
    '\t\t\t     ts_id);\n'
    '}\n'
    '\n'
    '\n'
    'static void __dvb_add_update_transponders'
)

# ── dvb-scan.c: __dvb_add_update_transponders に SYS_DVBC_ANNEX_B を追加 ─────
patch('lib/libdvbv5/dvb-scan.c',
    '\tcase SYS_DVBC_ANNEX_A:\n'
    '\tcase SYS_DVBC_ANNEX_C:\n'
    '\t\tdvb_table_nit_descriptor_handler(',
    '\tcase SYS_DVBC_ANNEX_A:\n'
    '\tcase SYS_DVBC_ANNEX_B:\n'
    '\tcase SYS_DVBC_ANNEX_C:\n'
    '\t\tdvb_table_nit_descriptor_handler('
)

# ── dvb-fe.c: __dvb_fe_get_parms — ATSCMH と DVBC_ANNEX_B を分離 ─────────────
patch('lib/libdvbv5/dvb-fe.c',
    '\tcase SYS_ATSC:\n'
    '\tcase SYS_ATSCMH:\n'
    '\tcase SYS_DVBC_ANNEX_B:\n'
    '\t\tdvb_fe_store_parm(&parms->p, DTV_MODULATION, v3_parms.u.vsb.modulation);\n'
    '\t\tbreak;\n'
    '\tcase SYS_DVBT:',
    '\tcase SYS_ATSC:\n'
    '\tcase SYS_ATSCMH:\n'
    '\t\tdvb_fe_store_parm(&parms->p, DTV_MODULATION, v3_parms.u.vsb.modulation);\n'
    '\t\tbreak;\n'
    '\tcase SYS_DVBC_ANNEX_B:\n'
    '\t\tdvb_fe_store_parm(&parms->p, DTV_SYMBOL_RATE, v3_parms.u.qam.symbol_rate);\n'
    '\t\tdvb_fe_store_parm(&parms->p, DTV_MODULATION, v3_parms.u.vsb.modulation);\n'
    '\t\tbreak;\n'
    '\tcase SYS_DVBT:'
)

# ── dvb-fe.c: dvb_setup_delsys_default — DVB-C/S2 で JP 時に arib-std-b24 ───
patch('lib/libdvbv5/dvb-fe.c',
    '\tswitch (p->current_sys) {\n'
    '\tcase SYS_ISDBT:\n'
    '\t\t/* Set country code. */',
    '\tswitch (p->current_sys) {\n'
    '\tcase SYS_DVBC_ANNEX_A:\n'
    '\tcase SYS_DVBC_ANNEX_B:\n'
    '\tcase SYS_DVBC_ANNEX_C:\n'
    '\tcase SYS_DVBS2:\n'
    '\t\tcc = COUNTRY_UNKNOWN;\n'
    '\t\tdvb_fe_retrieve_parm(p, DTV_COUNTRY_CODE, &cc);\n'
    '\t\tif (cc == COUNTRY_UNKNOWN && parms->country != COUNTRY_UNKNOWN) {\n'
    '\t\t\tcc = parms->country;\n'
    '\t\t\tdvb_fe_store_parm(p, DTV_COUNTRY_CODE, cc);\n'
    '\t\t}\n'
    '\t\tswitch (cc) {\n'
    '\t\tcase JP:\n'
    '\t\t\tp->default_charset = "arib-std-b24";\n'
    '\t\t\tbreak;\n'
    '\t\tdefault:\n'
    '\t\t\tbreak;\n'
    '\t\t}\n'
    '\t\tbreak;\n'
    '\tcase SYS_ISDBT:\n'
    '\t\t/* Set country code. */'
)

# ── dvb-fe.c: __dvb_fe_set_parms — ATSCMH と DVBC_ANNEX_B を分離 ─────────────
patch('lib/libdvbv5/dvb-fe.c',
    '\tcase SYS_ATSC:\n'
    '\tcase SYS_ATSCMH:\n'
    '\tcase SYS_DVBC_ANNEX_B:\n'
    '\t\tdvb_fe_retrieve_parm(&tmp_parms.p, DTV_MODULATION, &v3_parms.u.vsb.modulation);\n'
    '\t\tbreak;\n'
    '\tcase SYS_DVBT:',
    '\tcase SYS_ATSC:\n'
    '\tcase SYS_ATSCMH:\n'
    '\t\tdvb_fe_retrieve_parm(&tmp_parms.p, DTV_MODULATION, &v3_parms.u.vsb.modulation);\n'
    '\t\tbreak;\n'
    '\tcase SYS_DVBC_ANNEX_B:\n'
    '\t\tdvb_fe_retrieve_parm(&tmp_parms.p, DTV_SYMBOL_RATE, &v3_parms.u.qam.symbol_rate);\n'
    '\t\tdvb_fe_retrieve_parm(&tmp_parms.p, DTV_MODULATION, &v3_parms.u.vsb.modulation);\n'
    '\t\tbreak;\n'
    '\tcase SYS_DVBT:'
)

# ── dvb-fe.c: dvb_fe_get_event — ATSCMH と DVBC_ANNEX_B を分離 ──────────────
patch('lib/libdvbv5/dvb-fe.c',
    '\tcase SYS_ATSC:\n'
    '\tcase SYS_ATSCMH:\n'
    '\tcase SYS_DVBC_ANNEX_B:\n'
    '\t\tdvb_fe_retrieve_parm(&parms->p, DTV_MODULATION, &event.parameters.u.vsb.modulation);\n'
    '\t\tbreak;\n'
    '\tcase SYS_DVBT:',
    '\tcase SYS_ATSC:\n'
    '\tcase SYS_ATSCMH:\n'
    '\t\tdvb_fe_retrieve_parm(&parms->p, DTV_MODULATION, &event.parameters.u.vsb.modulation);\n'
    '\t\tbreak;\n'
    '\tcase SYS_DVBC_ANNEX_B:\n'
    '\t\tdvb_fe_retrieve_parm(&parms->p, DTV_SYMBOL_RATE, &event.parameters.u.qam.symbol_rate);\n'
    '\t\tdvb_fe_retrieve_parm(&parms->p, DTV_MODULATION, &event.parameters.u.vsb.modulation);\n'
    '\t\tbreak;\n'
    '\tcase SYS_DVBT:'
)

print("All patches applied.")
PYEOF
RUN python3 /tmp/patch.py

RUN meson setup build \
        --prefix=/usr \
        --buildtype=release \
        -Dqv4l2=disabled \
        -Dqvidcap=disabled \
        -Dv4l-wrappers=false \
        -Dv4l-plugins=false \
        -Dbpf=disabled \
        -Ddoxygen-doc=disabled \
    && DESTDIR=/out ninja -C build install


# ── Stage 2: libaribb25 のビルド（arib-b1/b25-stream-test を含む）────────────
#
# tsukumijima/libaribb25 は以下を提供する:
#   - arib-b1-stream-test  : スカパー！プレミアムサービス(ARIB STD-B1)のデスクランブル
#   - arib-b25-stream-test : 地上/BS/CS デジタル放送(ARIB STD-B25)のデスクランブル
#   - libaribb1 / libaribb25 : 各共有ライブラリ
#
FROM ${ARCH}debian:bookworm-slim AS aribb25-build

RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates git \
        cmake make \
        gcc g++ \
        pkg-config \
        libpcsclite-dev \
    && rm -rf /var/lib/apt/lists/*

RUN git clone --depth 1 https://github.com/tsukumijima/libaribb25.git /src

WORKDIR /src
RUN cmake -B build -DCMAKE_INSTALL_PREFIX=/usr \
    && make -j"$(nproc)" -C build \
    && DESTDIR=/out make -C build install


# ── Stage 3: dantto4k のビルド（tsduck は公式 .deb を使用）─────────────────────
#
# tsduck のソースビルドは20〜30分かかるため、GitHub Releases の公式 .deb を使用する。
# tsduck-dev: ヘッダ・静的ライブラリ（dantto4k のビルドに必要）
# tsduck:     共有ライブラリ（ランタイムに必要）
#
FROM ${ARCH}debian:bookworm-slim AS dantto4k-build

RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates curl git \
        make g++ python3 \
        libssl-dev zlib1g-dev libcurl4-openssl-dev \
        libpcsclite-dev pkgconf \
    && rm -rf /var/lib/apt/lists/*

# tsduck-dev の最新 .deb を GitHub Releases からダウンロードしてインストール
RUN curl -fsSL https://api.github.com/repos/tsduck/tsduck/releases/latest \
      | python3 -c "
import sys, json
assets = json.load(sys.stdin)['assets']
url = next(a['browser_download_url'] for a in assets
           if 'tsduck-dev' in a['name'] and 'debian12_amd64' in a['name'])
print(url)
" \
      | xargs curl -fsSL -o /tmp/tsduck-dev.deb \
    && dpkg -i /tmp/tsduck-dev.deb \
    && rm /tmp/tsduck-dev.deb

RUN git clone --depth 1 https://github.com/nekohkr/dantto4k.git /src \
    && cd /src && git submodule update --init --recursive --depth 1 \
    && make -j"$(nproc)" && make install


# ── Stage 4: Mirakurun (BS4K対応fork) のビルド ───────────────────────────────
#
# b-meu/Mirakurun: upstream chinachu/Mirakurun に BS4K 差分を適用した fork。
# 主な追加機能:
#   - チャンネルタイプ BS4K の追加
#   - tuners.yml に mmtsDecoder オプションを追加
#   - BS4K の EPG をサービス単位で取得
#   - BS4K チャンネルでは MPEG-2 TS デコーダをスキップ（dantto4k が処理）
#
FROM ${ARCH}node:24.16.0-bookworm AS build

WORKDIR /app
ENV DOCKER=YES NODE_ENV=production
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends build-essential ca-certificates git && \
    rm -rf /var/lib/apt/lists/*

RUN git clone --depth 1 https://github.com/b-meu/Mirakurun.git .
RUN npm ci --include=dev && npm run build && npm ci --omit=dev


# ── Stage 5: ランタイム ───────────────────────────────────────────────────────
FROM ${ARCH}node:24.16.0-bookworm-slim

WORKDIR /app
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
        ca-certificates curl python3 \
        pcscd \
        libpcsclite-dev \
        libccid \
        pcsc-tools \
        libudev1 \
        libssl3 \
        libcurl4 \
        zlib1g \
    && rm -rf /var/lib/apt/lists/*

# v4l-utils 1.32.0 (JP パッチ適用済み): libdvbv5 と dvb-tools
COPY --from=v4l-build /out/usr /usr

# libaribb25: arib-b1-stream-test / arib-b25-stream-test
COPY --from=aribb25-build /out/usr /usr

# tsduck ランタイム .deb（dantto4k が依存する共有ライブラリ）
RUN curl -fsSL https://api.github.com/repos/tsduck/tsduck/releases/latest \
      | python3 -c "
import sys, json
assets = json.load(sys.stdin)['assets']
url = next(a['browser_download_url'] for a in assets
           if a['name'].startswith('tsduck_') and 'debian12_amd64' in a['name'])
print(url)
" \
      | xargs curl -fsSL -o /tmp/tsduck.deb \
    && dpkg -i /tmp/tsduck.deb \
    && rm /tmp/tsduck.deb \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# dantto4k バイナリ（tsduck-dev でビルド済み）
COPY --from=dantto4k-build /usr/local/bin/dantto4k /usr/local/bin/dantto4k

RUN ldconfig

COPY --from=build /app /app
CMD ["./docker/container-init.sh"]
EXPOSE 40772 9229
