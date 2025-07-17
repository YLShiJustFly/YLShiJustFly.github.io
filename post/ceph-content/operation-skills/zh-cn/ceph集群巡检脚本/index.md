### 🛠️ 脚本功能特点

#### 全面的检查项目
- ✅ **集群连接状态** - 验证Ceph集群可达性
- ✅ **健康状态分析** - HEALTH_OK/WARN/ERR详细分析
- ✅ **Monitor节点** - 仲裁状态和节点可用性
- ✅ **OSD存储节点** - 在线状态和使用率分布
- ✅ **PG状态** - Placement Group完整性检查
- ✅ **存储容量** - 使用率监控和告警
- ✅ **性能监控** - 慢请求和IO统计
- ✅ **网络连接** - 集群网络状态检查

#### 高级功能
- 🔔 **多种告警方式** - 邮件、Webhook
- 📊 **多种输出格式** - 彩色终端、JSON、日志文件
- ⚙️ **灵活配置** - 自定义阈值和检查级别
- 🎯 **详细模式** - 深度诊断信息

### 🚀 立即使用

#### 基本使用
```bash
# 赋予执行权限
chmod +x ceph_cluster_health_monitor.sh

# 基本健康检查
./ceph_cluster_health_monitor.sh

# 详细检查并保存日志
./ceph_cluster_health_monitor.sh --detailed --log

# JSON格式输出
./ceph_cluster_health_monitor.sh --json --output health_report.json
```

#### 告警配置
```bash
# 启用邮件和Webhook告警
./ceph_cluster_health_monitor.sh \
  --email admin@company.com \
  --webhook https://your-webhook.url \
  --usage-warn 75 \
  --usage-crit 85
```

#### 定时检查
```bash
# 添加到crontab实现定时检查
# 每小时检查一次
0 * * * * /path/to/ceph_cluster_health_monitor.sh --quiet --log

# 每天详细检查并发送报告
0 6 * * * /path/to/ceph_cluster_health_monitor.sh --detailed --email admin@company.com
```

### 具体脚本

```
#!/bin/bash
#
# 支持全面的集群状态检查和告警
#

set -euo pipefail

# 全局配置
SCRIPT_NAME="Ceph集群健康监控"
VERSION="2.0"
TIMESTAMP=$(date +"%Y-%m-%d %H:%M:%S")
LOG_FILE="/var/log/ceph_health_check.log"

# 颜色和图标定义
readonly RED='[0;31m'
readonly GREEN='[0;32m'
readonly YELLOW='[1;33m'
readonly BLUE='[0;34m'
readonly CYAN='[0;36m'
readonly PURPLE='[0;35m'
readonly NC='[0m'

readonly CHECK_MARK="✅"
readonly WARNING_SIGN="⚠️"
readonly ERROR_SIGN="❌"
readonly INFO_SIGN="ℹ️"
readonly ROCKET="🚀"

# 选项默认值
DETAILED=false
JSON_OUTPUT=false
QUIET=false
SAVE_LOG=false
EMAIL_ALERT=""
WEBHOOK_URL=""
THRESHOLD_USAGE=80
THRESHOLD_CRITICAL=90

# 统计变量
TOTAL_CHECKS=0
PASSED_CHECKS=0
WARNING_CHECKS=0
FAILED_CHECKS=0

# 使用帮助
show_help() {
    cat << EOF
$SCRIPT_NAME v$VERSION

用法: $0 [选项]

基本选项:
  -h, --help              显示此帮助信息
  -v, --version           显示版本信息
  -d, --detailed          启用详细模式
  -q, --quiet             静默模式（仅显示错误）
  -j, --json              JSON格式输出

输出选项:
  -l, --log               保存日志到文件
  -o, --output FILE       指定输出文件
  --no-color              禁用颜色输出

告警选项:
  -e, --email EMAIL       告警邮件地址
  -w, --webhook URL       Webhook告警URL
  --usage-warn NUM        使用率警告阈值 (默认: 80%)
  --usage-crit NUM        使用率严重阈值 (默认: 90%)

示例:
  $0                                    # 基本健康检查
  $0 -d -l                             # 详细检查并保存日志
  $0 -j -o report.json                 # JSON格式输出到文件
  $0 -e admin@company.com -w http://webhook.url # 启用告警

EOF
}

# 参数解析
parse_arguments() {
    while [[ $# -gt 0 ]]; do
        case $1 in
            -h|--help) show_help; exit 0 ;;
            -v|--version) echo "$SCRIPT_NAME v$VERSION"; exit 0 ;;
            -d|--detailed) DETAILED=true ;;
            -q|--quiet) QUIET=true ;;
            -j|--json) JSON_OUTPUT=true ;;
            -l|--log) SAVE_LOG=true ;;
            -o|--output) OUTPUT_FILE="$2"; shift ;;
            --no-color) RED=''; GREEN=''; YELLOW=''; BLUE=''; CYAN=''; PURPLE=''; NC='' ;;
            -e|--email) EMAIL_ALERT="$2"; shift ;;
            -w|--webhook) WEBHOOK_URL="$2"; shift ;;
            --usage-warn) THRESHOLD_USAGE="$2"; shift ;;
            --usage-crit) THRESHOLD_CRITICAL="$2"; shift ;;
            *) echo "未知参数: $1"; show_help; exit 1 ;;
        esac
        shift
    done
}

# 日志函数
log_message() {
    local level="$1"
    local message="$2"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    
    if [ "$SAVE_LOG" = true ]; then
        echo "[$timestamp] [$level] $message" >> "$LOG_FILE"
    fi
    
    if [ "$QUIET" = false ] || [ "$level" = "ERROR" ]; then
        case $level in
            "INFO") echo -e "${BLUE}${INFO_SIGN} $message${NC}" ;;
            "SUCCESS") echo -e "${GREEN}${CHECK_MARK} $message${NC}" ;;
            "WARNING") echo -e "${YELLOW}${WARNING_SIGN} $message${NC}" ;;
            "ERROR") echo -e "${RED}${ERROR_SIGN} $message${NC}" ;;
            "SECTION") echo -e "${CYAN}=== $message ===${NC}" ;;
        esac
    fi
}

# 检查依赖
check_dependencies() {
    local deps=("ceph")
    local missing=()
    
    [ "$JSON_OUTPUT" = true ] && deps+=("jq")
    [ -n "$EMAIL_ALERT" ] && deps+=("mail")
    
    for dep in "${deps[@]}"; do
        if ! command -v "$dep" &> /dev/null; then
            missing+=("$dep")
        fi
    done
    
    if [ ${#missing[@]} -ne 0 ]; then
        log_message "ERROR" "缺少依赖: ${missing[*]}"
        exit 1
    fi
}

# 增加检查计数
increment_check() {
    ((TOTAL_CHECKS++))
    case "$1" in
        "PASS") ((PASSED_CHECKS++)) ;;
        "WARN") ((WARNING_CHECKS++)) ;;
        "FAIL") ((FAILED_CHECKS++)) ;;
    esac
}

# JSON输出辅助函数
json_start() {
    [ "$JSON_OUTPUT" = true ] && echo "{"
}

json_end() {
    [ "$JSON_OUTPUT" = true ] && echo "}"
}

json_add() {
    [ "$JSON_OUTPUT" = true ] && echo "  \"$1\": $2,"
}

# 1. 集群连接和基本状态
check_cluster_connection() {
    log_message "SECTION" "集群连接检查"
    
    if timeout 10 ceph status &> /dev/null; then
        log_message "SUCCESS" "成功连接到Ceph集群"
        increment_check "PASS"
        
        local cluster_id=$(ceph fsid 2>/dev/null)
        local ceph_version=$(ceph version | awk '{print $3}')
        
        log_message "INFO" "集群ID: $cluster_id"
        log_message "INFO" "Ceph版本: $ceph_version"
        
        [ "$JSON_OUTPUT" = true ] && json_add "cluster_connection" '"success"'
        return 0
    else
        log_message "ERROR" "无法连接到Ceph集群"
        increment_check "FAIL"
        [ "$JSON_OUTPUT" = true ] && json_add "cluster_connection" '"failed"'
        return 1
    fi
}

# 2. 集群健康状态
check_cluster_health() {
    log_message "SECTION" "集群健康状态检查"
    
    local health_output
    health_output=$(ceph health detail 2>/dev/null)
    local health_status=$(echo "$health_output" | head -1 | awk '{print $1}')
    
    case "$health_status" in
        "HEALTH_OK")
            log_message "SUCCESS" "集群健康状态: OK"
            increment_check "PASS"
            ;;
        "HEALTH_WARN")
            log_message "WARNING" "集群健康状态: 警告"
            [ "$DETAILED" = true ] && echo "$health_output"
            increment_check "WARN"
            ;;
        "HEALTH_ERR")
            log_message "ERROR" "集群健康状态: 错误"
            echo "$health_output"
            increment_check "FAIL"
            ;;
        *)
            log_message "ERROR" "无法获取集群健康状态"
            increment_check "FAIL"
            ;;
    esac
    
    [ "$JSON_OUTPUT" = true ] && json_add "health_status" "\"$health_status\""
}

# 3. Monitor状态检查
check_monitor_health() {
    log_message "SECTION" "Monitor节点检查"
    
    local mon_stat
    mon_stat=$(ceph mon stat 2>/dev/null)
    
    # 检查仲裁状态
    local quorum_size
    quorum_size=$(ceph quorum_status --format=json 2>/dev/null | jq '.quorum | length' 2>/dev/null || echo "0")
    
    local total_mons
    total_mons=$(echo "$mon_stat" | grep -o '[0-9]\+ mons' | cut -d' ' -f1)
    
    if [ "$quorum_size" -gt 0 ] && [ "$quorum_size" -eq "$((total_mons))" ]; then
        log_message "SUCCESS" "所有 $total_mons 个Monitor节点正常运行"
        increment_check "PASS"
    elif [ "$quorum_size" -gt "$((total_mons / 2))" ]; then
        log_message "WARNING" "Monitor仲裁正常，但 $((total_mons - quorum_size)) 个节点离线"
        increment_check "WARN"
    else
        log_message "ERROR" "Monitor仲裁失败！仅 $quorum_size/$total_mons 节点在线"
        increment_check "FAIL"
    fi
    
    if [ "$DETAILED" = true ]; then
        log_message "INFO" "Monitor详细状态:"
        ceph mon dump 2>/dev/null | head -10
    fi
    
    [ "$JSON_OUTPUT" = true ] && json_add "monitor_quorum" "$quorum_size"
}

# 4. OSD状态检查
check_osd_health() {
    log_message "SECTION" "OSD存储节点检查"
    
    local osd_stat
    osd_stat=$(ceph osd stat 2>/dev/null)
    
    local total_osds=$(echo "$osd_stat" | grep -o '[0-9]\+ osds' | cut -d' ' -f1)
    local up_osds=$(echo "$osd_stat" | grep -o '[0-9]\+ up' | cut -d' ' -f1)
    local in_osds=$(echo "$osd_stat" | grep -o '[0-9]\+ in' | cut -d' ' -f1)
    
    log_message "INFO" "OSD状态: $up_osds/$total_osds 运行中, $in_osds/$total_osds 服务中"
    
    if [ "$up_osds" -eq "$total_osds" ] && [ "$in_osds" -eq "$total_osds" ]; then
        log_message "SUCCESS" "所有OSD节点正常"
        increment_check "PASS"
    elif [ "$up_osds" -gt "$((total_osds * 80 / 100))" ]; then
        log_message "WARNING" "$((total_osds - up_osds)) 个OSD离线"
        increment_check "WARN"
        
        if [ "$DETAILED" = true ]; then
            log_message "INFO" "离线OSD详情:"
            ceph osd tree | grep -E "down|out" | head -5
        fi
    else
        log_message "ERROR" "严重: $((total_osds - up_osds)) 个OSD离线"
        increment_check "FAIL"
    fi
    
    # 检查OSD使用率分布
    if [ "$DETAILED" = true ]; then
        log_message "INFO" "OSD使用率分布:"
        ceph osd df | head -10
        
        # 检查使用率不均衡
        local max_usage min_usage
        max_usage=$(ceph osd df --format=json 2>/dev/null | jq -r '.nodes[] | .utilization' | sort -nr | head -1 | cut -d. -f1 2>/dev/null || echo "0")
        min_usage=$(ceph osd df --format=json 2>/dev/null | jq -r '.nodes[] | .utilization' | sort -n | head -1 | cut -d. -f1 2>/dev/null || echo "0")
        
        if [ "$((max_usage - min_usage))" -gt 20 ]; then
            log_message "WARNING" "OSD使用率不均衡: 最高${max_usage}% 最低${min_usage}%"
        fi
    fi
    
    [ "$JSON_OUTPUT" = true ] && json_add "osd_total" "$total_osds"
    [ "$JSON_OUTPUT" = true ] && json_add "osd_up" "$up_osds"
}

# 5. PG状态检查
check_pg_health() {
    log_message "SECTION" "Placement Group检查"
    
    local pg_stat
    pg_stat=$(ceph pg stat 2>/dev/null)
    
    # 检查active+clean的PG数量
    local total_pgs active_clean_pgs
    total_pgs=$(echo "$pg_stat" | grep -o '[0-9]\+ pgs' | cut -d' ' -f1)
    active_clean_pgs=$(echo "$pg_stat" | grep -o '[0-9]\+ active+clean' | cut -d' ' -f1 || echo "0")
    
    if [ "$active_clean_pgs" -eq "$total_pgs" ]; then
        log_message "SUCCESS" "所有 $total_pgs 个PG状态正常 (active+clean)"
        increment_check "PASS"
    else
        local problem_pgs=$((total_pgs - active_clean_pgs))
        log_message "WARNING" "$problem_pgs/$total_pgs 个PG状态异常"
        increment_check "WARN"
        
        if [ "$DETAILED" = true ]; then
            log_message "INFO" "异常PG状态分布:"
            ceph pg dump summary 2>/dev/null | grep -v "active+clean" | head -10
        fi
    fi
    
    # 检查stuck PG
    local stuck_count
    stuck_count=$(ceph pg dump stuck 2>/dev/null | wc -l)
    if [ "$stuck_count" -gt 1 ]; then  # 减去header
        log_message "WARNING" "发现 $((stuck_count-1)) 个stuck PG"
    fi
    
    [ "$JSON_OUTPUT" = true ] && json_add "pg_total" "$total_pgs"
    [ "$JSON_OUTPUT" = true ] && json_add "pg_active_clean" "$active_clean_pgs"
}

# 6. 存储容量检查
check_storage_capacity() {
    log_message "SECTION" "存储容量检查"
    
    local df_output
    df_output=$(ceph df 2>/dev/null)
    
    local usage_percent
    usage_percent=$(echo "$df_output" | grep "TOTAL" | awk '{print $4}' | sed 's/%//' | head -1)
    usage_percent=${usage_percent:-0}
    
    local total_capacity used_capacity available_capacity
    total_capacity=$(echo "$df_output" | grep "TOTAL" | awk '{print $1}' | head -1)
    used_capacity=$(echo "$df_output" | grep "TOTAL" | awk '{print $2}' | head -1)
    available_capacity=$(echo "$df_output" | grep "TOTAL" | awk '{print $3}' | head -1)
    
    log_message "INFO" "存储容量: 总计 $total_capacity, 已用 $used_capacity, 可用 $available_capacity"
    
    if [ "$usage_percent" -lt "$THRESHOLD_USAGE" ]; then
        log_message "SUCCESS" "存储使用率 $usage_percent% 正常"
        increment_check "PASS"
    elif [ "$usage_percent" -lt "$THRESHOLD_CRITICAL" ]; then
        log_message "WARNING" "存储使用率 $usage_percent% 需要关注"
        increment_check "WARN"
    else
        log_message "ERROR" "存储使用率 $usage_percent% 严重告警！"
        increment_check "FAIL"
    fi
    
    if [ "$DETAILED" = true ]; then
        log_message "INFO" "存储池详细使用情况:"
        echo "$df_output" | grep -A 20 "POOLS:"
    fi
    
    [ "$JSON_OUTPUT" = true ] && json_add "storage_usage_percent" "$usage_percent"
}

# 7. 性能和IO检查
check_performance() {
    if [ "$DETAILED" = true ]; then
        log_message "SECTION" "性能状态检查"
        
        # 检查慢请求
        local slow_requests
        slow_requests=$(ceph health detail | grep -c "slow requests" || echo "0")
        
        if [ "$slow_requests" -eq 0 ]; then
            log_message "SUCCESS" "无慢请求"
            increment_check "PASS"
        else
            log_message "WARNING" "检测到 $slow_requests 个慢请求"
            increment_check "WARN"
        fi
        
        # IO统计
        log_message "INFO" "当前IO统计:"
        timeout 3 ceph iostat 2>/dev/null | head -5 || log_message "INFO" "无法获取IO统计"
        
        [ "$JSON_OUTPUT" = true ] && json_add "slow_requests" "$slow_requests"
    fi
}

# 8. 网络和连接检查
check_network_connectivity() {
    if [ "$DETAILED" = true ]; then
        log_message "SECTION" "网络连接检查"
        
        local public_network cluster_network
        public_network=$(ceph config get mon public_network 2>/dev/null || echo "未配置")
        cluster_network=$(ceph config get mon cluster_network 2>/dev/null || echo "未配置")
        
        log_message "INFO" "公共网络: $public_network"
        log_message "INFO" "集群网络: $cluster_network"
        
        # 检查网络分离
        if ceph health detail | grep -q "clock skew"; then
            log_message "WARNING" "检测到时钟偏差，可能存在网络问题"
            increment_check "WARN"
        else
            increment_check "PASS"
        fi
    fi
}

# 发送告警
send_alert() {
    local subject="$1"
    local message="$2"
    
    # 邮件告警
    if [ -n "$EMAIL_ALERT" ]; then
        echo "$message" | mail -s "$subject" "$EMAIL_ALERT" 2>/dev/null || \
            log_message "ERROR" "发送邮件告警失败"
    fi
    
    # Webhook告警
    if [ -n "$WEBHOOK_URL" ]; then
        curl -X POST -H "Content-Type: application/json" \
             -d "{\"subject\":\"$subject\",\"message\":\"$message\"}" \
             "$WEBHOOK_URL" 2>/dev/null || \
            log_message "ERROR" "发送Webhook告警失败"
    fi
}

# 生成最终报告
generate_report() {
    log_message "SECTION" "健康检查总结"
    
    local overall_status="HEALTHY"
    local alert_message=""
    
    if [ "$FAILED_CHECKS" -gt 0 ]; then
        overall_status="CRITICAL"
        alert_message="Ceph集群存在 $FAILED_CHECKS 个严重问题！"
        log_message "ERROR" "$alert_message"
    elif [ "$WARNING_CHECKS" -gt 0 ]; then
        overall_status="WARNING"
        alert_message="Ceph集群存在 $WARNING_CHECKS 个警告！"
        log_message "WARNING" "$alert_message"
    else
        log_message "SUCCESS" "Ceph集群运行正常！"
    fi
    
    # 统计摘要
    log_message "INFO" "检查统计: 总计 $TOTAL_CHECKS, 通过 $PASSED_CHECKS, 警告 $WARNING_CHECKS, 失败 $FAILED_CHECKS"
    
    # 发送告警
    if [ "$overall_status" != "HEALTHY" ] && { [ -n "$EMAIL_ALERT" ] || [ -n "$WEBHOOK_URL" ]; }; then
        send_alert "Ceph集群健康告警 - $overall_status" "$alert_message"
    fi
    
    # JSON输出
    if [ "$JSON_OUTPUT" = true ]; then
        json_add "overall_status" "\"$overall_status\""
        json_add "total_checks" "$TOTAL_CHECKS"
        json_add "passed_checks" "$PASSED_CHECKS"
        json_add "warning_checks" "$WARNING_CHECKS"
        json_add "failed_checks" "$FAILED_CHECKS"
        json_add "timestamp" "\"$TIMESTAMP\""
    fi
    
    # 返回适当的退出码
    case "$overall_status" in
        "HEALTHY") return 0 ;;
        "WARNING") return 1 ;;
        "CRITICAL") return 2 ;;
    esac
}

# 主函数
main() {
    # 解析参数
    parse_arguments "$@"
    
    # 输出头部信息
    if [ "$QUIET" = false ]; then
        echo -e "${PURPLE}${ROCKET} $SCRIPT_NAME v$VERSION${NC}"
        echo -e "${BLUE}检查时间: $TIMESTAMP${NC}"
        echo -e "${BLUE}执行用户: $(whoami)${NC}"
        echo ""
    fi
    
    # 检查依赖
    check_dependencies
    
    # JSON输出开始
    [ "$JSON_OUTPUT" = true ] && json_start
    
    # 执行检查
    check_cluster_connection || exit 3
    check_cluster_health
    check_monitor_health
    check_osd_health
    check_pg_health
    check_storage_capacity
    check_performance
    check_network_connectivity
    
    # 生成报告
    local exit_code
    generate_report
    exit_code=$?
    
    # JSON输出结束
    [ "$JSON_OUTPUT" = true ] && json_end
    
    # 保存输出到文件
    if [ -n "${OUTPUT_FILE:-}" ]; then
        # 重新运行并保存到文件
        "$0" "$@" --no-color > "$OUTPUT_FILE" 2>&1
        log_message "INFO" "报告已保存到: $OUTPUT_FILE"
    fi
    
    return $exit_code
}

# 信号处理
trap 'log_message "ERROR" "脚本被中断"; exit 130' INT TERM

# 执行主函数
main "$@"

```
