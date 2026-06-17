CronJob là gì? Tưởng tượng cho dễ hiểu
Ở phần trước, chúng ta đã biết:

Job giống như một người thợ được thuê để làm một công việc một lần rồi thôi (ví dụ: sơn hàng rào).

Bây giờ, hãy tưởng tượng bạn cần người thợ đó làm một công việc có tính chất lặp đi lặp lại theo lịch trình. Ví dụ: "Cứ 6 giờ sáng mỗi ngày, hãy tưới cây trong vườn."

Bạn sẽ không thể mỗi sáng thức dậy lúc 6 giờ để gọi người thợ đó được. Thay vào đó, bạn cần một thứ gì đó có thể tự động "gọi" người thợ đó dậy làm việc đúng giờ, mỗi ngày.

CronJob trong Kubernetes chính là cái ĐỒNG HỒ BÁO THỨC đó.

Nó không tự mình làm việc.

Nhiệm vụ duy nhất của nó là "reo" lên vào một thời điểm đã được định sẵn.

Mỗi khi nó "reo", nó sẽ tạo ra một Job (gọi người thợ dậy) để thực hiện công việc.

Tóm lại: CronJob là một bộ lập lịch, có nhiệm vụ tạo ra các Job một cách định kỳ. Mối quan hệ của chúng là:

CronJob (Cái đồng hồ báo thức) → tạo ra → Job (Người thợ) → tạo ra → Pod (Công cụ để làm việc)

Phân tích file YAML ví dụ
Ví dụ này sẽ in ra thời gian hiện tại và một lời chào mỗi phút.

YAML

apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  # --- Phần 1: Lịch trình "reo chuông" ---
  schedule: "* * * * *" # <-- Chạy mỗi phút

  # --- Phần 2: Khuôn mẫu cho Job sẽ được tạo ra ---
  jobTemplate: # <-- Đây là bản thiết kế của "người thợ"
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.28
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
Các điểm đáng chú ý trong YAML:
kind: CronJob: Khai báo đây là một CronJob.

spec.schedule: Đây là trái tim của CronJob, xác định lịch trình. Nó sử dụng cú pháp Cron của Unix, rất mạnh mẽ.

# ┌───────────── phút (0 - 59)
# │ ┌───────────── giờ (0 - 23)
# │ │ ┌───────────── ngày trong tháng (1 - 31)
# │ │ │ ┌───────────── tháng (1 - 12)
# │ │ │ │ ┌───────────── ngày trong tuần (0 - 6) (Chủ Nhật đến Thứ 7)
# │ │ │ │ │
# * * * * *
* * * * *: Chạy mỗi phút.

0 * * * *: Chạy vào phút thứ 0 của mỗi giờ (tức là mỗi giờ một lần).

0 2 * * *: Chạy vào lúc 2:00 sáng mỗi ngày.

0 5 * * 1: Chạy vào lúc 5:00 sáng mỗi thứ Hai.

@daily: Một cách viết tắt cho 0 0 * * *.

spec.jobTemplate: Đây là một bản thiết kế y hệt như một Job bình thường. Mọi thứ bạn đã học về cách cấu hình một Job (như backoffLimit, template của Pod...) đều được đặt vào bên trong mục này. CronJob sẽ dùng cái khuôn này để tạo ra một Job mới mỗi khi đến giờ.

Các chính sách điều khiển quan trọng
CronJob cung cấp một vài tùy chọn rất hữu ích để kiểm soát hành vi của nó.

1. concurrencyPolicy (Chính sách chạy đồng thời)
Vấn đề: Điều gì sẽ xảy ra nếu tác vụ sao lưu của ngày hôm qua (Job A) chạy quá lâu và đến giờ chạy tác vụ của ngày hôm nay (Job B) mà nó vẫn chưa xong?

Allow (mặc định): "Không quan tâm, cứ tạo Job B đi." Kết quả là bạn có thể có 2 Job sao lưu chạy cùng lúc, có thể gây xung đột.

Forbid: "Khoan đã, Job A chưa xong. Bỏ qua lần chạy của Job B này." Nó sẽ không tạo Job mới nếu Job cũ vẫn còn đang chạy.

Replace: "Dừng ngay Job A đang chạy và thay thế nó bằng Job B mới." Job mới sẽ được ưu tiên.

2. startingDeadlineSeconds (Hạn chót để bắt đầu)
Vấn đề: Lịch của bạn là chạy sao lưu lúc 2 giờ sáng, nhưng cả cluster bị sập để bảo trì từ 1 giờ đến 4 giờ sáng. Khi cluster hoạt động trở lại, có nên chạy bù tác vụ lúc 2 giờ sáng không?

Trường này định nghĩa một "thời gian ân hạn". Ví dụ startingDeadlineSeconds: 3600 (1 giờ).

Ý nghĩa: "Nếu một Job bị lỡ lịch trình của nó, hãy cố gắng chạy nó ngay khi có thể. Nhưng nếu nó đã bị trễ quá 3600 giây so với lịch trình ban đầu, thì hãy bỏ qua nó luôn."

Điều này rất hữu ích cho các tác vụ nhạy cảm về thời gian. Một bản sao lưu lúc 4 giờ sáng có thể vẫn hữu ích, nhưng một bản sao lưu lúc 10 giờ sáng cho lịch trình 2 giờ sáng thì có thể đã quá muộn và vô dụng.

3. successfulJobsHistoryLimit & failedJobsHistoryLimit
Vấn đề: CronJob tạo ra rất nhiều Job theo thời gian. Nếu không dọn dẹp, chúng sẽ gây "rác" hệ thống.

Các trường này cho phép bạn giữ lại một số lượng Job cũ để kiểm tra.

successfulJobsHistoryLimit: 3 (mặc định): Giữ lại 3 Job gần nhất đã hoàn thành thành công.

failedJobsHistoryLimit: 1 (mặc định): Giữ lại 1 Job gần nhất đã bị thất bại.

Các Job cũ hơn sẽ tự động bị xóa, giúp cluster luôn gọn gàng.

Tóm tắt
Nếu Job là một "người thợ" cho một công việc cụ thể, thì CronJob chính là "người quản lý với một cuốn lịch làm việc". Người quản lý này không tự làm việc, mà chỉ cần đến đúng ngày đúng giờ, anh ta sẽ tạo ra một phiếu yêu cầu công việc (một Job) và giao cho người thợ thực hiện.

Đây là công cụ tiêu chuẩn trong Kubernetes để tự động hóa các tác vụ định kỳ như sao lưu, tạo báo cáo, dọn dẹp dữ liệu...