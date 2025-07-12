---
title: 'A.I Power-Up Case Study: Lập trình tool thu thập báo cáo tài chính với Gemini trong 30 phút'
date: 2025-07-11 16:30:00 +0700
categories: [Hands-On]
tags: [A.I, Crawler, Financial Statement]
pin: false
---

# Dẫn nhập

Dạo gần đây mình có tự học thêm về kỹ năng đọc và phân tích báo cáo tài chính của doanh nghiệp bằng cách xem video Youtube, sau đó lại kết hợp với Gemini thì mình nhận ra một số điểm như sau:

1. Nguồn tài liệu của những video Youtube thường là những báo cáo còn mới, số lượng ít và các chỉ số thường rất đẹp. Nếu muốn phân tích được nhiều hơn thì cần phải đọc nhiều báo cáo tài chính của nhiều công ty khác nhau, thuộc những nhóm ngành khác nhau
2. Khi mình upload file lên cho Gemini, thật bất ngờ là nó hoàn toàn có thể phân tích tốt, những nhóm chỉ số đều được tính toán và đưa ra phân tích rất hay. Mình hoàn toàn được mở mang chỉ bằng vài câu prompt

Từ đó mình tin rằng kỹ năng này có thể học từ A.I thay vì đăng ký một lớp học. Dĩ nhiên có thể sẽ không thay được hoàn toàn khóa học do con người giảng dạy, nhưng mình tin đa số kiến thức đều có thể học từ A.I trước

Thế là mình đã quyết định tìm cách thu thập nhiều nhất có thể các báo cáo tài chính của các công ty có niêm yết trên sàn chứng khoán Việt Nam, sau đó sẽ đưa vào cho A.I phân tích.

Và thật bất ngờ, từ lúc mình bắt đầu quyết định viết một tool cơ bản để làm việc này cho tới lúc làm xong, kết quả chỉ vỏn vẹn… 30 phút. Ban đầu mình còn nghĩ phải mất mấy tiếng tranh thủ lúc rảnh rỗi cơ.

Bài blog này là một case study chi tiết của mình, từ những bước đầu tiên tìm kiếm nguồn dữ liệu cho đến việc tận dụng sức mạnh của AI để tự động hóa quy trình. Mình sử dụng Gemini 2.5 Flash cho project này

# Bước 1. Tìm nơi chứa đầy đủ báo cáo tài chính

Bước đầu tiên và quan trọng nhất là xác định nguồn dữ liệu đáng tin cậy và đầy đủ.

Thế là mình đi hỏi A.I

```
Tôi có thể tìm và download toàn bộ các báo cáo tài chính của các công ty niêm yết trên sàn chứng khoán ở đâu
```

Kết quả sẽ trả về nhiều trang web, sẽ có một số trang không dùng được, vì Gemini có khả năng bịa câu trả lời. Sau nhiều lần thử thì mình tìm ra trang web [https://cafef.vn/du-lieu.chn](https://cafef.vn/du-lieu.chn) . Chỉ cần nhập mã cổ phiếu một công ty sau đó kéo xuống phần Tải BCTC là sẽ có danh sách các BCTC từ cũ tới mới. Thế là xong bước 1

![](https://images2.imgbox.com/66/83/hSxXe2vU_o.png)

# Bước 2. Giải mã cấu trúc API

Với nguồn dữ liệu đã có, bước tiếp theo là tìm hiểu cách mình có thể tự động truy cập chúng. Mình đã dành thời gian **phân tích cấu trúc API** của trang web. Cụ thể, mình đã tìm hiểu cách các biến số trong URL thay đổi để lấy được báo cáo tài chính của từng mã công ty. Chi tiết mình xin phép không ghi ra đây

# Bước 3. Dùng A.I để viết code

Khi đã có cái nhìn tổng quan về API, tiếp theo mình cần một công cụ để tự động hóa việc download các file báo cáo tài chính. Cụ thể gồm các bước:

1. Một vòng lặp để request tuần tự các API, với mỗi tổ hợp mã cổ phiếu và năm báo cáo là một lần request
2. Sau khi request sẽ nhận lại response tương ứng, phân tích response để lấy ra được đường dẫn download file
3. Từ danh sách đường dẫn đó, tiếp tục download file pdf và sắp xếp chúng theo từng thư mục tương ứng với mỗi biến số tên công ty và năm ở bước 1. Tổ chức thư mục theo dạng <mã công ty>/<năm>

Và đây là mẫu prompt:

```
Cho một danh sách mã công ty. Hãy viết một chương trình bằng Python thực hiện các bước như sau:

1/ Viết một đoạn code python để request url này: <url here>. Thay các biến số sym là mã công ty từ danh sách đã cho và year từ 2000 đến 2025.

2/ Sau khi request sẽ nhận lại response tương ứng. Dưới đây là một response mẫu khi request vào url bước 1. Hãy viết code để lấy được thông tin tất cả đường dẫn để download file pdf. 

3/ Từ danh sách đường dẫn đó hãy tiếp tục download file pdf và sắp xếp chúng theo từng thư mục tương ứng với mỗi biến số tên công ty và năm ở bước 1. Tổ chức thư mục theo dạng <mã công ty>/<năm>

==========================================

<Paste đoạn response ở bước 2 vào đây>
```

Xong, A.I đã generate ra cho mình đoạn code đầy đủ mình chỉ việc chạy. Cái hay của nó là in ra các log đầy đủ để mình tiện theo dõi nữa

```python
import requests
from bs4 import BeautifulSoup
import os
import time

def fetch_financial_reports(company_codes, start_year, end_year):
    """
    Gửi yêu cầu đến Cafef.vn để lấy dữ liệu báo cáo tài chính cho các công ty và năm đã cho.

    Args:
        company_codes (list): Danh sách các mã công ty (ví dụ: ['FPT', 'HPG']).
        start_year (int): Năm bắt đầu.
        end_year (int): Năm kết thúc.

    Returns:
        dict: Một dictionary chứa phản hồi HTML, được tổ chức theo {mã công ty: {năm: html_content}}.
    """
    base_url = "<url here>"
    all_responses = {}

    for company_code in company_codes:
        all_responses[company_code] = {}
        for year in range(start_year, end_year + 1):
            params = {
                "sym": company_code,
                "type": 1,
                "year": year
            }
            try:
                print(f"Đang lấy dữ liệu cho {company_code} - Năm {year}...")
                response = requests.get(base_url, params=params, timeout=10) # Thêm timeout
                response.raise_for_status()  # Nâng lỗi cho mã trạng thái HTTP xấu (4xx hoặc 5xx)
                all_responses[company_code][year] = response.text
                print(f"-> Đã lấy dữ liệu thành công cho {company_code} - Năm {year}")
            except requests.exceptions.Timeout:
                print(f"Lỗi timeout khi lấy dữ liệu cho {company_code} - Năm {year}. Bỏ qua.")
            except requests.exceptions.RequestException as e:
                print(f"Lỗi khi lấy dữ liệu cho {company_code} - Năm {year}: {e}. Bỏ qua.")
            time.sleep(0.5) # Dừng một chút để tránh quá tải server
    return all_responses

def extract_pdf_links(html_content):
    """
    Trích xuất tất cả các đường dẫn PDF từ nội dung HTML đã cho.

    Args:
        html_content (str): Nội dung HTML của trang.

    Returns:
        list: Một danh sách các đường dẫn URL của tệp PDF.
    """
    soup = BeautifulSoup(html_content, 'html.parser')
    pdf_links = []
    # Tìm tất cả các thẻ 'a' có thuộc tính 'href' kết thúc bằng '.pdf'
    # và nằm trong div có class 'treeview'
    for a_tag in soup.select('div.treeview a[href$=".pdf"]'):
        href = a_tag.get('href')
        if href:
            pdf_links.append(href)
    return pdf_links

def download_pdfs(all_pdf_links):
    """
    Tải xuống các tệp PDF từ các URL đã cho và lưu chúng vào cấu trúc thư mục.

    Args:
        all_pdf_links (dict): Một dictionary chứa các đường dẫn PDF,
                              được tổ chức theo {mã công ty: {năm: [links]}}.
    """
    for company_code, years_data in all_pdf_links.items():
        for year, pdf_links in years_data.items():
            # Tạo thư mục đích: <mã công ty>/<năm>
            download_dir = os.path.join(company_code, str(year))
            os.makedirs(download_dir, exist_ok=True)
            print(f"\nĐang xử lý thư mục: {download_dir}")

            if not pdf_links:
                print(f"Không tìm thấy tệp PDF nào để tải xuống cho {company_code} - Năm {year}.")
                continue

            for pdf_url in pdf_links:
                try:
                    file_name = os.path.basename(pdf_url).split('?')[0] # Loại bỏ các tham số URL nếu có
                    file_path = os.path.join(download_dir, file_name)

                    # Kiểm tra xem tệp đã tồn tại chưa
                    if os.path.exists(file_path):
                        print(f"  Tệp '{file_name}' đã tồn tại. Bỏ qua.")
                        continue

                    print(f"  Đang tải xuống: '{file_name}'...")
                    response = requests.get(pdf_url, stream=True, timeout=30) # Thêm timeout
                    response.raise_for_status()

                    with open(file_path, 'wb') as f:
                        for chunk in response.iter_content(chunk_size=8192):
                            f.write(chunk)
                    print(f"  -> Đã tải xuống thành công: '{file_name}'")
                except requests.exceptions.Timeout:
                    print(f"  Lỗi timeout khi tải xuống {pdf_url}. Bỏ qua.")
                except requests.exceptions.RequestException as e:
                    print(f"  Lỗi khi tải xuống {pdf_url}: {e}. Bỏ qua.")
                except Exception as e:
                    print(f"  Lỗi không xác định khi xử lý {pdf_url}: {e}. Bỏ qua.")
                time.sleep(0.5) # Dừng một chút giữa các lần tải xuống

if __name__ == "__main__":
    # Danh sách mã công ty bạn muốn thu thập dữ liệu
    company_codes_to_collect = ["FPT", "HPG", "VCB"] # Bạn có thể thêm hoặc bớt các mã công ty ở đây
    
    # Phạm vi năm bạn muốn thu thập báo cáo
    start_year = 2000
    end_year = 2025 # Lưu ý: Dữ liệu thực tế có thể không có đến năm 2025

    print("--- Bắt đầu quá trình thu thập báo cáo tài chính ---")
    print(f"Mã công ty: {', '.join(company_codes_to_collect)}")
    print(f"Phạm vi năm: {start_year} - {end_year}")

    # Bước 1: Lấy dữ liệu HTML từ Cafef.vn
    html_responses = fetch_financial_reports(company_codes_to_collect, start_year, end_year)

    # Bước 2: Trích xuất các liên kết PDF từ nội dung HTML
    all_pdf_links = {}
    print("\n--- Bắt đầu trích xuất liên kết PDF ---")
    for company_code, years_data in html_responses.items():
        all_pdf_links[company_code] = {}
        for year, html_content in years_data.items():
            links = extract_pdf_links(html_content)
            if links:
                all_pdf_links[company_code][year] = links
                print(f"  Tìm thấy {len(links)} liên kết PDF cho {company_code} - Năm {year}.")
            else:
                print(f"  Không tìm thấy liên kết PDF nào cho {company_code} - Năm {year}.")

    # Bước 3: Tải xuống và tổ chức các tệp PDF vào thư mục
    if all_pdf_links:
        print("\n--- Bắt đầu tải xuống và tổ chức tệp PDF ---")
        download_pdfs(all_pdf_links)
        print("\n--- Quá trình tải xuống và tổ chức báo cáo tài chính đã hoàn tất! ---")
    else:
        print("\nKhông có liên kết PDF nào được tìm thấy để tải xuống.")
```

Gemini không quên nhắc mình cần phải cài đặt thư viện `requests` và `beautifulsoup4`

Lại còn biết dừng lại một chút giữa các lần download, những cái này bình thường mình chỉ thêm vào sau khi dev hoàn chỉnh project

Mình đã chạy thử và hoàn toàn chạy ngon lành

# Bước 4. Dùng A.I để lấy danh sách mã cổ phiếu

Sau khi đã chạy code thử với 3 mã cổ phiếu, bây giờ mình cần mở rộng bài toán với toàn bộ mã cổ phiếu của thị trường chứng khoán Việt Nam năm 2025. Cái này thì mình hỏi Google Search thôi. Kết quả mình tìm thấy trang web này có chứa thông tin đầy đủ [https://anfin.vn/blog/danh-sach-ma-co-phieu-theo-nganh/](https://anfin.vn/blog/danh-sach-ma-co-phieu-theo-nganh/)

Mình thử nhờ Gemini request trực tiếp vào trang này và lấy thông tin nhưng có vẻ không được. Nên mình đã copy toàn bộ HTML của trang này thành 1 file và hỏi Gemini

```
đọc file này và trích xuất thông tin thành file csv gồm tên ngành, tên công ty, mã cổ phiếu, sàn niêm yết
```

Kết quả trả về:

```python
import csv
from bs4 import BeautifulSoup

def extract_stock_data_from_html_file(html_file_path):
    """
    Đọc một file HTML, phân tích cú pháp để trích xuất danh sách mã cổ phiếu theo ngành.

    Args:
        html_file_path (str): Đường dẫn đến file HTML chứa dữ liệu.

    Returns:
        list: Một danh sách các dictionaries, mỗi dictionary đại diện cho một hàng dữ liệu
              với các khóa 'Ngành', 'Tên công ty', 'Mã cổ phiếu', 'Sàn niêm yết'.
              Trả về danh sách rỗng nếu có lỗi hoặc không tìm thấy dữ liệu.
    """
    try:
        with open(html_file_path, 'r', encoding='utf-8') as f:
            html_content = f.read()
    except FileNotFoundError:
        print(f"Lỗi: Không tìm thấy file tại đường dẫn '{html_file_path}'")
        return []
    except Exception as e:
        print(f"Lỗi khi đọc file HTML: {e}")
        return []

    soup = BeautifulSoup(html_content, 'html.parser')
    all_stock_data = []

    # Tìm tất cả các tiêu đề ngành (thường là <h2> chứa <strong>tên ngành</strong>)
    # và các bảng đi kèm với chúng.
    # Cấu trúc của trang Anfin.vn sử dụng <figure class="table"> chứa <table>.

    # Các tiêu đề ngành thường nằm trong thẻ <h2> có chứa <strong>
    industry_sections = soup.find_all(['h2', 'figure', 'div'], class_=lambda x: x and ('table' in x or 'style_body_table__bS10F' in x))

    current_industry = "Chưa xác định"

    for element in soup.find_all(lambda tag: tag.name in ['h2', 'figure']):
        if element.name == 'h2':
            # Trích xuất tên ngành từ thẻ <h2>
            strong_tag = element.find('strong')
            if strong_tag:
                raw_industry_name = strong_tag.get_text(strip=True)
                # Làm sạch tên ngành: loại bỏ số thứ tự và dấu chấm, "Ngành"
                if '.' in raw_industry_name:
                    parts = raw_industry_name.split('.', 1)
                    if len(parts) > 1:
                        raw_industry_name = parts[1].strip()
                if "Ngành" in raw_industry_name:
                    current_industry = raw_industry_name.replace("Ngành", "").strip()
                else:
                     current_industry = raw_industry_name.strip()

        elif element.name == 'figure' and 'class' in element.attrs and 'table' in element['class']:
            # Trong mỗi figure (bảng), tìm thẻ <table>
            table = element.find('table')
            if table:
                rows = table.find_all('tr')
                # Bỏ qua hàng đầu tiên (tiêu đề bảng)
                for row in rows[1:]:
                    cols = row.find_all('td')
                    if len(cols) >= 4:
                        # Dữ liệu theo thứ tự: #, Tên công ty, Mã cổ phiếu, Sàn niêm yết
                        # Chúng ta cần: Tên công ty, Mã cổ phiếu, Sàn niêm yết
                        company_name = cols[1].get_text(strip=True)
                        stock_code = cols[2].get_text(strip=True)
                        exchange = cols[3].get_text(strip=True)

                        # Bỏ qua hàng nếu không có mã cổ phiếu hoặc là hàng tiêu đề phụ
                        if not stock_code or stock_code.lower() == "mã cổ phiếu":
                            continue

                        all_stock_data.append({
                            'Ngành': current_industry,
                            'Tên công ty': company_name,
                            'Mã cổ phiếu': stock_code,
                            'Sàn niêm yết': exchange
                        })
    return all_stock_data

def save_to_csv(data, filename="danh_sach_ma_co_phieu_anfin.csv"):
    """
    Lưu dữ liệu vào một file CSV.

    Args:
        data (list): Danh sách các dictionaries chứa dữ liệu.
        filename (str): Tên file CSV để lưu.
    """
    if not data:
        print("Không có dữ liệu để lưu vào CSV.")
        return

    # Lấy các khóa từ dictionary đầu tiên để làm header cho CSV
    headers = list(data[0].keys())

    try:
        with open(filename, 'w', newline='', encoding='utf-8') as csvfile:
            writer = csv.DictWriter(csvfile, fieldnames=headers)
            writer.writeheader()
            writer.writerows(data)
        print(f"Dữ liệu đã được lưu thành công vào file '{filename}'")
    except IOError as e:
        print(f"Lỗi khi ghi file CSV: {e}")

if __name__ == "__main__":
    html_input_file = "anfin_ma_cp.html" # Tên file HTML đã tải lên
    csv_output_file = "danh_sach_co_phieu_anfin.csv"

    print(f"Đang đọc dữ liệu từ file: {html_input_file}")
    stock_data = extract_stock_data_from_html_file(html_input_file)

    if stock_data:
        print(f"Đã trích xuất được {len(stock_data)} dòng dữ liệu.")
        save_to_csv(stock_data, csv_output_file)
    else:
        print("Không thể trích xuất dữ liệu mã cổ phiếu từ file đã cho.")
```

Thế là đã có ngay 1 file CSV với những thông tin cần thiết, bây giờ chỉ việc ráp lại với đoạn code ở bước 3 là xong project

# Lesson Learned

Và đó là toàn bộ quá trình từ lúc mình lên ý tưởng đến lúc dev xong một dự án với sự hỗ trợ của A.I. Hy vọng case study này sẽ giúp ích cho mọi người
