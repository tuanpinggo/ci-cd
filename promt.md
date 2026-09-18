đã cài xong jenkins trên server 1, hướng dẫn tôi các bước tiếp theo

- cài harbor trên server 1
- setup các agent cho jenkins, thực hiện các bước
    - checkout commit mới nhất trên nhánh (jenkins có màn hình chọn nhánh main hay develop)
    - Test / lint / security scan
    - chạy build image 
    - Scan + ký image
    - Push image lên harbor
    - cập nhật GitOps repository

Viết hướng dẫn này vào setup-jenkins.md

