# get nlbwmon output to prometheus metrics for grafana

instructions adapted from 
- https://forum.openwrt.org/t/nlbwmon-prometheus-lua-file-looking-for-help/154820

1. install these packages
  - nlbwmon
  - prometheus-node-exporter-lua

2. add following to crontab (UI >-> system -> scheduled tasks)
```
* * * * * nlbw -c csv -g ip,mac -o ip | tr -d '"' | tail -n +2 > /tmp/nlbwmon.out
```

3. create the following file with this text /usr/lib/lua/prometheus-collectors/nlbwmon.lua
```
local nlbwstat = {
    "mac",
    "ip",
    "conns",
    "rx_bytes",
    "rx_pkts",
    "tx_bytes",
    "tx_pkts"
}

local pattern = "(%w+:%w+:%w+:%w+:%w+:%w+)%s+(%d+.%d+.%d+.%d+)%s+(%d+)%s+(%d+)%s+(%d+)%s+(%d+)%s+(%d+)"

local function scrape()
  local nlbw_table = {}
  for line in io.lines("/tmp/nlbwmon.out") do
    local t = {string.match(line, pattern)}
    if #t == 7 then
      nlbw_table[t[1]] = t
    end
  end

  -- create a new metric
  nlbw_metric = metric("nlbwmon_stats", "counter")

  -- iterate through the nlbw_table and add the values to the metric
  for mac, nlbw in pairs(nlbw_table) do
    nlbw_metric({
      mac = nlbw[1],
      ip = nlbw[2],
      conns = nlbw[3],
      rx_bytes = nlbw[4],
      rx_pkts = nlbw[5],
      tx_bytes = nlbw[6],
      tx_pkts = nlbw[7]
    }, nlbw[7])
  end
end

return { scrape = scrape }

```
