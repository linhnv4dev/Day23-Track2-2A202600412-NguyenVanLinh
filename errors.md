## 1. submission/screenshots/dashboard-overview.png

  1. Mở http://localhost:3000
  2. Login nếu hỏi:
    - user: admin
    - password: admin
  3. Vào menu Dashboards.
  4. Mở dashboard AI Service Overview (Day 23).
  5. Góc phải trên chọn time range: Last 15 minutes.
  6. Bấm refresh.
  7. LỖI KHÔNG THẤY có data:
    - Request Rate
    - Latency P50/P95/P99
    - Error Rate
    - AI Quality Score
    - Token Throughput
    - In-Flight Requests


## 2. submission/screenshots/slo-burn-rate.png

  1. Mở Grafana http://localhost:3000
  2. Mở dashboard SLO Burn Rate (Day 23).
  3. Chọn time range Last 6 hours hoặc Last 15 minutes.
  4. Bấm refresh.
  5. LỖI KHÔNG thấy:
    - Error Budget Remaining có số.
    - Burn Rate có line hoặc giá trị 0.
    - Active Alerts table có thể trống cũng được.


## 3. submission/screenshots/cost-and-tokens.png

  1. Tạo thêm data ngay trước khi chụp:

  for i in {1..50}; do curl -s -X POST
  http://localhost:8000/predict -H 'Content-Type: 
  application/json' -d '{"prompt":"hello cost token 
  dashboard"}' > /dev/null; done

  2. Đợi 30–45 giây.
  3. Mở dashboard Cost & Tokens (Day 23).
  4. Chọn time range Last 15 minutes.
  5. Bấm refresh.
  6. LỖI KHÔNG thấy:
    - Token Throughput có input/output.
    - Estimated $/hr có số khác 0.
    - Eval Quality Score có data.

## 6. submission/screenshots/slack-resolved.png

  Sau khi đã chụp firing, chạy:

  docker start day23-app

  Đợi khoảng 60–90 giây.

  Sau đó trong Slack:

  1. Vẫn ở channel #observability.
  2. LỖI KHÔNG Tìm thấy message resolved, thường có dấu
  resolved/green/check hoặc status resolved.





///
Nguyên nhân bạn vẫn thấy No Data thường là:
  1. App đang tắt sau lần chụp alert.
  2. Chưa tạo đủ request sau khi app restart.
  3. Mở dashboard quá sớm, Prometheus chưa scrape đủ sample.
  4. Time range không bao gồm lúc vừa tạo data.
  5. Grafana cache chưa refresh.

  Trước khi chụp lại ảnh dashboard

  Chạy đúng các lệnh này:

  docker start day23-app
  make verify

  Nếu make verify pass, tạo data mới:

  for i in {1..80}; do curl -s -X POST
  http://localhost:8000/predict -H 'Content-Type: 
  application/json' -d '{"prompt":"hello observability dashboard 
  data for grafana panels"}' > /dev/null; sleep 1; done

  Sau đó đợi thêm 30–45 giây để Prometheus scrape.

  Trong Grafana, nếu vẫn thấy No Data:
  - Bấm nút refresh góc phải trên.
  - Chọn time range Last 15 minutes.
  - Hard refresh browser: Ctrl + F5.
  - Nếu vẫn chưa được, mở dashboard rồi chọn lại datasource
  Prometheus nếu panel báo datasource lỗi.

  ---
  1. Chụp lại dashboard-overview.png
  
  1. Mở: http://localhost:3000
  2. Vào Dashboards.
  3. Mở AI Service Overview (Day 23).
  4. Góc phải trên chọn Last 15 minutes.
  5. Bấm refresh.
  6. Kiểm tra các panel sau đã có data:
    - Request Rate (RPS) by status
    - Latency P50 / P95 / P99
    - Error Rate (last 5m) — có thể là 0, vẫn hợp lệ.
    - AI Quality Score
    - Token Throughput
    - In-Flight Requests — có thể là 0, vẫn hợp lệ.
  7. Chụp và lưu:

  submission/screenshots/dashboard-overview.png

  ---
  2. Chụp lại slo-burn-rate.png
  
  1. Mở dashboard SLO Burn Rate (Day 23).
  2. Chọn time range Last 15 minutes hoặc Last 6 hours.
  3. Bấm refresh.
  4. Panel hợp lệ khi thấy:
    - Error Budget Remaining (%) có số.
    - Burn Rate có line hoặc giá trị 0.
    - Active Alerts có thể trống, không sao.
  5. Chụp và lưu:

  submission/screenshots/slo-burn-rate.png

  ---
  3. Chụp lại cost-and-tokens.png
  
  Ngay trước khi chụp, tạo data mới:

  for i in {1..80}; do curl -s -X POST
  http://localhost:8000/predict -H 'Content-Type: 
  application/json' -d '{"prompt":"hello cost token dashboard"}' >
   /dev/null; sleep 1; done

  Đợi 30–45 giây.

  Sau đó:

  1. Mở dashboard Cost & Tokens (Day 23).
  2. Chọn Last 15 minutes.
  3. Bấm refresh.
  4. Kiểm tra:
    - Token Throughput có input/output.
    - Estimated $/hr có số khác 0.
    - Eval Quality Score có data.
  5. Chụp và lưu:

  submission/screenshots/cost-and-tokens.png

  ---
  4. Chụp lại Slack resolved
  
  Lỗi của bạn là chưa thấy message resolved. Làm theo đúng thứ tự
  này.

  Bước A — Tạo alert firing và giữ app tắt

  Chạy:

  HOLD_FOR_SCREENSHOT=1 bash scripts/trigger-alert.sh

  Đợi tới khi terminal hiện:

  Alert is firing. Capture Alertmanager and Slack screenshots now.
  When done, run: docker start day23-app

  Lúc này đừng start app ngay.

  Chụp Slack firing trước nếu cần:

  submission/screenshots/slack-firing.png

  Bước B — Restore app để Alertmanager gửi resolved

  Sau khi đã thấy firing trong Slack, chạy:

  docker start day23-app

  Kiểm tra app đã lên:

  curl http://localhost:8000/healthz

  Nếu trả về OK/JSON health là được.

  Bước C — Đợi resolved đủ lâu

  Đợi 2–3 phút, không chỉ 60 giây.

  Lý do:
  - Prometheus rule ServiceDown có for: 1m.
  - Prometheus scrape/evaluate theo chu kỳ.
  - Alertmanager có group_interval: 1m.
  - Slack message resolved có thể chậm thêm một chút.

  Sau đó mở Slack channel #observability, tìm message có trạng
  thái:
  - resolved
  - hoặc icon xanh/check
  - hoặc title kiểu resolved/recovered.

  Chụp và lưu:

  submission/screenshots/slack-resolved.png

  Nếu vẫn không thấy Slack resolved, kiểm tra Alertmanager đã hết
  alert chưa:

  curl -s http://localhost:9093/api/v2/alerts

  Nếu trả về [] nghĩa là alert đã resolve ở Alertmanager; khi đó
  đợi Slack thêm 1 phút rồi refresh Slack.

  ---
  Kiểm tra cuối cùng
  
  Sau khi chụp xong, chạy:

  docker start day23-app
  make verify
  git status --short

  Hiện tại verify đã pass 12/12; bạn chỉ cần chụp lại các ảnh còn
  lỗi theo hướng dẫn trên.

✻ Crunched for 3m 11s

※ recap: Goal: finish the observability lab and retake 
  screenshots. Next action: run the metric-generation curl loop, 
  wait 30–45 seconds, then reopen Grafana dashboards with “Last 15
   minutes” and refresh before capturing. (disable recaps in 
  /config)

