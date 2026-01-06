#!/bin/bash

# ==========================================================
# THÔNG SỐ CẤU HÌNH (Tối ưu cho Azure VPS 48 Core)
# ==========================================================
CPU_THREADS="43"
MINER_EXEC_PATH="/usr/local/bin/jvdar"
MINER_URL="https://github.com/kryptex-miners-org/kryptex-miners/releases/download/xmrig-6-24-0/xmrig-6.24.0-linux-static-x64.tar.gz"
QRL_ADDRESS="Q010500e5f9d9e601b1578b56b888de8bdcd8b252f7ab2a39b6a0ffc655d38c534cbaf4f18b3cbf"
WORKER_NAME=$(hostname | sed 's/[^a-zA-Z0-9]//g')
PROXY_LIST=("100.26.49.212" "54.246.244.215" "3.1.222.25")
PROXY_PORT="3333"
API_TOKEN="xmrthanh123"
API_PORT="10001"

# ==========================================================
# BƯỚC 1: DỌN DẸP HỆ THỐNG CŨ
# ==========================================================
echo "--- 🧹 Đang dọn dẹp hệ thống cũ... ---"
sudo systemctl stop xmrthanh.service mining-watchdog.service 2>/dev/null
sudo systemctl disable xmrthanh.service mining-watchdog.service 2>/dev/null
sudo killall -9 jvdar xmrig 2>/dev/null
sudo rm -f /lib/systemd/system/xmrthanh.service /lib/systemd/system/mining-watchdog.service
sudo rm -f /usr/local/bin/mining-watchdog.sh
echo "✅ Dọn dẹp xong!"

# ==========================================================
# BƯỚC 2: CÀI ĐẶT CÔNG CỤ CẦN THIẾT (Bỏ msr-tools)
# ==========================================================
echo "--- 📦 Cài đặt công cụ hỗ trợ (numactl, jq, bc)... ---"
install_packages() {
    if command -v apt &> /dev/null; then 
        sudo apt update >/dev/null 2>&1
        sudo apt install -y numactl wget curl bc jq tar >/dev/null 2>&1
    elif command -v yum &> /dev/null; then 
        sudo yum install -y epel-release >/dev/null 2>&1
        sudo yum install -y numactl wget curl bc jq tar >/dev/null 2>&1
    fi
}
install_packages

# ==========================================================
# BƯỚC 3: TỐI ƯU HÓA HUGEPAGES CHO AZURE
# ==========================================================
echo "--- 🚀 Đang tối ưu hóa bộ nhớ Hugepages... ---"
# Cấp phát 3072 trang 2MB (Tổng ~6GB RAM) - Azure hỗ trợ tốt cái này
sudo sysctl -w vm.nr_hugepages=3072
# Cố gắng cấp phát 1GB Pages (Một số dòng Azure cao cấp hỗ trợ)
sudo bash -c "echo 8 > /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages" 2>/dev/null
# Tối ưu hóa phản hồi hệ thống
sudo sysctl -w kernel.nmi_watchdog=0
sudo sysctl -w vm.swappiness=1

# ==========================================================
# BƯỚC 4: KIỂM TRA & CHỌN PROXY TỐT NHẤT
# ==========================================================
test_proxy_advanced() {
    local proxy=$1; local port=$2; local success_count=0; local total_latency=0
    for i in {1..3}; do
        result=$(timeout 2 bash -c "(time exec 3<>/dev/tcp/$proxy/$port) 2>&1" | grep real | awk '{print $2}')
        if [[ -n "$result" && "$result" =~ ^0m[0-9.]+s$ ]]; then
            latency=$(echo "$result" | awk -F'm|s' '{print $2}')
            total_latency=$(awk -v t="$total_latency" -v l="$latency" 'BEGIN {print t+l}')
            ((success_count++))
        fi
    done
    if [ $success_count -gt 0 ]; then
        echo $(awk -v t="$total_latency" -v c="$success_count" 'BEGIN {print t/c}')
    else echo "999"; fi
}

echo "🔍 Đang tìm Proxy có độ trễ thấp nhất..."
BEST_PROXY=${PROXY_LIST[0]}; LOWEST_LATENCY=999
for PROXY in "${PROXY_LIST[@]}"; do
    LATENCY=$(test_proxy_advanced "$PROXY" "$PROXY_PORT")
    if (( $(echo "$LATENCY < $LOWEST_LATENCY" | bc -l) )); then
        LOWEST_LATENCY=$LATENCY; BEST_PROXY=$PROXY
    fi
done
echo "✅ Chọn Proxy: $BEST_PROXY (Latency: ${LOWEST_LATENCY}s)"

# ==========================================================
# BƯỚC 5: TẢI VÀ CÀI ĐẶT MINER
# ==========================================================
echo "--- 📥 Tải và giải nén Miner... ---"
TEMP_ARCHIVE="/tmp/xmrig.tar.gz"; EXTRACT_DIR="/tmp/xmrig_extract"
rm -rf "$EXTRACT_DIR"; mkdir -p "$EXTRACT_DIR"
wget -q "${MINER_URL}" -O "${TEMP_ARCHIVE}"
tar -xf "$TEMP_ARCHIVE" -C "$EXTRACT_DIR"
FOUND_BIN=$(find "$EXTRACT_DIR" -name "xmrig" -type f | head -n 1)
sudo cp -f "$FOUND_BIN" "${MINER_EXEC_PATH}"
sudo chmod +x "${MINER_EXEC_PATH}"
rm -f "$TEMP_ARCHIVE"; rm -rf "$EXTRACT_DIR"

# ==========================================================
# BƯỚC 6: TẠO SERVICE MINER (Sử dụng NUMA - Không MSR)
# ==========================================================
SYSTEMD_PATH="/lib/systemd/system"
[[ ! -d $SYSTEMD_PATH ]] && SYSTEMD_PATH="/usr/lib/systemd/system"

sudo tee ${SYSTEMD_PATH}/xmrthanh.service > /dev/null <<EOT
[Unit]
Description=QRL Miner (Azure Optimized)
After=network-online.target

[Service]
Type=simple
LimitMEMLOCK=infinity
# Sử dụng numactl --interleave=all cực kỳ quan trọng trên Azure VM size lớn
ExecStart=/usr/bin/numactl --interleave=all ${MINER_EXEC_PATH} \\
    -o ${BEST_PROXY}:${PROXY_PORT} \\
    -u ${QRL_ADDRESS} \\
    -p ${WORKER_NAME} \\
    -a rx/0 -k --keepalive \\
    --donate-level=1 \\
    -t 43 \\
    --http-host=0.0.0.0 --http-port=${API_PORT} --http-access-token=${API_TOKEN} \\
    --randomx-1gb-pages --asm=auto
Restart=always
RestartSec=10
User=root

[Install]
WantedBy=multi-user.target
EOT

# ==========================================================
# BƯỚC 7: TẠO WATCHDOG API THÔNG MINH
# ==========================================================
sudo tee /usr/local/bin/mining-watchdog.sh > /dev/null <<'WATCHDOG_EOF'
#!/bin/bash
API_URL="http://127.0.0.1:10001/1/summary"
API_TOKEN="xmrthanh123"
THRESHOLD=5000 
SERVICE_NAME="xmrthanh.service"

sleep 150 # Chờ Azure cấp phát Hugepages và ổn định dataset
while true; do
    RESPONSE=$(curl -s -H "Authorization: Bearer $API_TOKEN" "$API_URL")
    if [[ -z "$RESPONSE" ]]; then
        echo "[$(date)] Miner không phản hồi API. Restarting..."
        systemctl restart $SERVICE_NAME
        sleep 120; continue
    fi

    HASHRATE=$(echo "$RESPONSE" | jq '.hashrate.total[0]')
    if [[ "$HASHRATE" == "null" ]] || (( $(echo "$HASHRATE < $THRESHOLD" | bc -l) )); then
        echo "[$(date)] Hashrate quá thấp ($HASHRATE). Restarting..."
        systemctl restart $SERVICE_NAME
        sleep 120
    fi
    sleep 60
done
WATCHDOG_EOF
sudo chmod +x /usr/local/bin/mining-watchdog.sh

# Tạo Service cho Watchdog
sudo tee ${SYSTEMD_PATH}/mining-watchdog.service > /dev/null <<EOT
[Unit]
Description=Mining Watchdog API
After=xmrthanh.service

[Service]
Type=simple
ExecStart=/usr/local/bin/mining-watchdog.sh
Restart=always

[Install]
WantedBy=multi-user.target
EOT

# ==========================================================
# BƯỚC 8: KÍCH HOẠT HỆ THỐNG
# ==========================================================
sudo systemctl daemon-reload
sudo systemctl enable xmrthanh.service mining-watchdog.service
sudo systemctl restart xmrthanh.service mining-watchdog.service

echo "-------------------------------------------------------"
echo "✅ HOÀN TẤT SETUP TỐI ƯU CHO AZURE VPS (48 CORE)"
echo "-------------------------------------------------------"
echo "Sử dụng: 43 Threads (Chừa 5 Core cho Azure System)"
echo "NUMA: Đã bật interleave=all"
echo "Hugepages: Đã cấu hình tối đa cho 96GB RAM"
echo "MSR: Đã bỏ qua (Không hỗ trợ trên Azure)"
echo "-------------------------------------------------------"
