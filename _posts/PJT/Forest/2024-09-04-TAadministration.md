---
published: true
title: "230909 TA administration 자동화"
last_modified_at: 2024-09-04
categories: Forest
tags: 

toc: true
toc_sticky: true
toc_label: "TA admin"

---

# 코드
---
```python
    # 기본 설정
    from selenium import webdriver
    from selenium.webdriver.common.by import By
    from selenium.webdriver.support.ui import WebDriverWait
    from selenium.webdriver.support import expected_conditions as EC
    from webdriver_manager.chrome import ChromeDriverManager  # webdriver_manager 모듈 임포트
    import time
    import os
    import gspread
    from oauth2client.service_account import ServiceAccountCredentials
    from xlrd import open_workbook
    from glob import glob

    # 다운로드 경로
    DOWNLOAD_DIR = '/Desktop/SDIJ_FSRT/TA/download-ta'

    # Chrome Driver 설정
    options = webdriver.ChromeOptions()
    # options.add_argument('headless')
    options.add_argument("disable-gpu")
    options.add_argument("lang=ko_KR")
    options.add_argument("user-agent=Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.100 Safari/537.36")

    options.add_experimental_option(
        "prefs",
        {
            "download.default_directory": DOWNLOAD_DIR,
            "download.prompt_for_download": False,
            "download.directory_upgrade": True,
            "safebrowsing.enabled": True
        })
    driver_path = '/Desktop/SDIJ_FSRT/TA/lib/webDriver/chromedriver'
    service = ChromeService(executable_path=ChromeDriverManager().install())
    driver = webdriver.Chrome(options=options, service=service)

    # 자동화 동작 순서
    try:
        # 1. 접속
        driver.get("https://...")
        driver.implicitly_wait(10)

        # 2. ID
        input_field_1 = driver.find_element(By.XPATH, "/html/body/div[1]/div/div/main/div/div/form/div/div[2]/label[1]/div/div[1]/div[1]/input")
        input_field_1.send_keys("")
        driver.implicitly_wait(10)

        # 3. PW
        input_field_2 = driver.find_element(By.XPATH, "/html/body/div[1]/div/div/main/div/div/form/div/div[2]/label[2]/div/div/div/input")
        input_field_2.send_keys("")
        driver.implicitly_wait(10)

        # 4. 로그인
        submit_button = driver.find_element(By.XPATH, "/html/body/div[1]/div/div/main/div/div/form/div/div[3]/button/span[2]")
        submit_button.click()
        driver.implicitly_wait(10)

        # 5. 다운로드
        download_button = driver.find_element(By.XPATH, "/html/body/div[1]/div/div[3]/main/div[2]/div[2]/div/div[2]/button")
        download_button.click()
        driver.implicitly_wait(10)
        time.sleep(15)

    finally:
        # 작업 완료 후 브라우저 닫기
        driver.quit()

    # GS 인증 및 스프레드시트 연결 설정
    scope = ['https://spreadsheets.google.com/feeds']
    json_file_name = '.json'
    credentials = ServiceAccountCredentials.from_json_keyfile_name(json_file_name, scope)
    gc = gspread.authorize(credentials)
    spreadsheet_url = 'https://docs.google.com/spreadsheets'
    worksheet_name = '시트1'  # 데이터를 업로드할 시트 이름
    worksheet = gc.open_by_url(spreadsheet_url).worksheet(worksheet_name)

    # 최신 .xls 파일 찾기
    download_dir = '/Users/rainyforest/Desktop/SDIJ_FSRT/TA/download-ta'
    xls_files = glob(download_dir + '/*.xls')
    if len(xls_files) >= 2:
        xls_files.sort(key=os.path.getctime, reverse=True)
        latest_xls_file = xls_files[0]
        second_latest_xls_file = xls_files[1]

        # 최신 .xls 파일 읽기
        latest_wb = open_workbook(latest_xls_file)
        latest_sheet = latest_wb.sheet_by_index(0)

        # 두번째 최신 .xls 파일 읽기
        second_latest_wb = open_workbook(second_latest_xls_file)
        second_latest_sheet = second_latest_wb.sheet_by_index(0)

        # 최신 파일에서 새로운 데이터 찾기
        new_data = []
        for row in range(1, latest_sheet.nrows):  # 1st row 제외하고 비교
            row_values = latest_sheet.row_values(row)
            if row_values not in [second_latest_sheet.row_values(i) for i in range(1, second_latest_sheet.nrows)]:
                new_data.append(row_values)

        # upload if there is new data
        if new_data:
            # add new data
            worksheet.insert_rows(new_data, row=2)  # adding under 1st row
            print(f"{len(new_data)}개의 새로운 데이터 업로드 완료")
        else:
            print("새로 추가된 데이터가 없습니다.")
    else:
        print("업로드할 .xls 파일이 충분하지 않습니다.")
```

