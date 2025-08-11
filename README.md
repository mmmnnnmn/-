# 교실 환기 판단
교실 환기 판단을 용이하게 해주는 웹사이트
classroom-ventilation/
├── index.html          # 프론트엔드 (웹페이지)
├── server_simple.py    # 백엔드 서버
└── README.md          # 설치/실행 가이드
# 서버 실행
python server_simple.py

# 브라우저에서 접속
http://localhost:5000
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>교실 환기 판단</title>
    <link href="https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@400;700&display=swap" rel="stylesheet">
    <style>
        /* CSS 스타일 (208줄) - 따뜻한 녹색 테마, 그라데이션, 반응형 디자인 */
        /* ... (위에서 보여드린 전체 CSS 코드) ... */
    </style>
</head>
<body>
    <header>
        <h1>교실 환기 판단</h1>
        <p>3시간 30분마다 환기 권장 – REHVA</p>
        <p>이른 새벽이나 밤에는 공기 정체 가능성 주의 – WHO, REHVA</p>
    </header>

    <main>
        <div class="status" id="ventilation-status">
            환기 가능 여부: <span class="loading">데이터 로딩 중...</span>
        </div>

        <div class="data-section">
            <div class="data-item" id="temperature">
                <span class="data-label">외부 온도:</span>
                <span class="data-value">- ℃</span>
            </div>
            <div class="data-item" id="pm10">
                <span class="data-label">PM10:</span>
                <span class="data-value">- ㎍/m³ (-)</span>
            </div>
            <div class="data-item" id="pm25">
                <span class="data-label">PM2.5:</span>
                <span class="data-value">- ㎍/m³ (-)</span>
            </div>
            <div class="data-item" id="wind-direction">
                <span class="data-label">풍향:</span>
                <span class="data-value">-</span>
            </div>
        </div>

        <div class="note" id="wind-note" style="display:none;">
            현재 풍향이 남동쪽이므로 교실 내 공기를 효과적으로 순환시킬 수 있습니다.
        </div>

        <button class="refresh-button" id="refresh-btn" onclick="loadAllData()">
            데이터 새로고침
        </button>

        <div class="last-updated" id="last-updated"></div>
    </main>

    <script>
        /* JavaScript 코드 (419줄) - API 호출, 데이터 처리, UI 업데이트 */
        /* ... (위에서 보여드린 전체 JavaScript 코드) ... */
    </script>
</body>
</html>
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import json
import urllib.request
import urllib.parse
import ssl
from http.server import HTTPServer, SimpleHTTPRequestHandler
from urllib.parse import urlparse
import logging
from datetime import datetime, timedelta

# 로깅 설정
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

class YonginVentilationHandler(SimpleHTTPRequestHandler):
    def do_GET(self):
        parsed_path = urlparse(self.path)
        
        if parsed_path.path == '/api/air-quality':
            self.handle_air_quality_api()
        elif parsed_path.path == '/api/weather':
            self.handle_weather_api()
        else:
            # 정적 파일 서빙
            super().do_GET()
    
    def handle_air_quality_api(self):
        """환경부 대기질 데이터 API"""
        try:
            # 경기도 API 사용
            api_key = "732b82f0bc6043679179f4f2bd173a4c"
            params = {
                'KEY': api_key,
                'Type': 'json',
                'pIndex': '1',
                'pSize': '50'
            }
            
            url = f"https://openapi.gg.go.kr/Sidoatmospolutnmesure?{urllib.parse.urlencode(params)}"
            
            with urllib.request.urlopen(url, timeout=10) as response:
                text = response.read().decode('utf-8')
                data = json.loads(text)
                
                # 경기도 API 데이터 변환
                if 'Sidoatmospolutnmesure' in data and len(data['Sidoatmospolutnmesure']) > 1:
                    rows = data['Sidoatmospolutnmesure'][1]['row']
                    
                    # 용인 지역 데이터 찾기
                    yongin_data = None
                    for row in rows:
                        if row.get('SIGUN_NM') and '용인' in str(row['SIGUN_NM']):
                            yongin_data = row
                            break
                    
                    if not yongin_data and rows:
                        yongin_data = rows[0]  # 용인 데이터가 없으면 첫 번째 사용
                    
                    if yongin_data:
                        # 표준 형식으로 변환
                        converted_data = {
                            'response': {
                                'body': {
                                    'items': [{
                                        'pm10Value': str(yongin_data.get('PM10', '45')),
                                        'pm25Value': str(yongin_data.get('PM25', '28')),
                                        'dataTime': str(yongin_data.get('MESURE_DT', datetime.now().strftime('%Y-%m-%d %H:%M'))),
                                        'stationName': str(yongin_data.get('MESURSTN_NM', '용인'))
                                    }]
                                }
                            }
                        }
                        
                        logger.info(f"실제 대기질 데이터: PM10={converted_data['response']['body']['items'][0]['pm10Value']}, PM2.5={converted_data['response']['body']['items'][0]['pm25Value']}")
                        self.send_json_response(converted_data)
                        return
                
                # 데이터가 없으면 데모 데이터
                raise ValueError("대기질 데이터를 찾을 수 없습니다")
                
        except Exception as e:
            logger.error(f"대기질 API 오류: {e}")
            # 데모 데이터 반환
            demo_data = {
                'response': {
                    'body': {
                        'items': [{
                            'pm10Value': '45',
                            'pm25Value': '28',
                            'dataTime': datetime.now().strftime('%Y-%m-%d %H:%M'),
                            'stationName': '용인(데모)'
                        }]
                    }
                },
                'demo_mode': True
            }
            self.send_json_response(demo_data)
    
    def handle_weather_api(self):
        """기상청 날씨 데이터 API"""
        try:
            # 실제 기상청 API 사용
            import urllib.parse as up
            api_key = up.unquote("725gBOkoKWzeMCmv34dnvWd8iET3ieIG6eyY3wy5tVPxPtoLbvxy%2BaFbqjrvCBudekbxmY%2FhpT067UwRBq2gcA%3D%3D")
            
            # 현재 시간 기준으로 기상청 API 파라미터 설정
            now = datetime.now()
            target_time = now
            
            # 현재 분이 40분 미만이면 1시간 전으로 설정 (기상청 API 특성)
            if now.minute < 40:
                target_time = now - timedelta(hours=1)
            
            base_date = target_time.strftime('%Y%m%d')
            base_time = target_time.strftime('%H00')
            
            params = {
                'serviceKey': api_key,
                'numOfRows': '100',
                'pageNo': '1',
                'dataType': 'JSON',
                'base_date': base_date,
                'base_time': base_time,
                'nx': '63',  # 용인시 격자 X 좌표
                'ny': '120'  # 용인시 격자 Y 좌표
            }
            
            url = f"http://apis.data.go.kr/1360000/VilageFcstInfoService_2.0/getUltraSrtNcst?{urllib.parse.urlencode(params)}"
            
            # SSL 컨텍스트 설정
            ssl_context = ssl.create_default_context()
            ssl_context.check_hostname = False
            ssl_context.verify_mode = ssl.CERT_NONE
            
            with urllib.request.urlopen(url, context=ssl_context, timeout=10) as response:
                text = response.read().decode('utf-8')
                data = json.loads(text)
                
                # 기상청 API 응답 확인
                if ('response' in data and 'body' in data['response'] and 
                    'items' in data['response']['body']):
                    
                    logger.info(f"실제 기상청 데이터: {base_date} {base_time}")
                    self.send_json_response(data)
                    return
                else:
                    raise ValueError("기상청 API 응답에 필요한 데이터가 없습니다")
            
        except Exception as e:
            logger.error(f"기상청 API 오류: {e}")
            # 오류 시 데모 데이터 반환
            demo_weather = {
                'response': {
                    'body': {
                        'items': {
                            'item': [
                                {'category': 'T1H', 'obsrValue': '22'},  # 온도
                                {'category': 'VEC', 'obsrValue': '135'}  # 풍향 (남동)
                            ]
                        }
                    }
                },
                'demo_mode': True
            }
            
            logger.info("기상청 API 오류로 데모 데이터 사용: 22°C, 남동풍")
            self.send_json_response(demo_weather)
    
    def send_json_response(self, data):
        """JSON 응답 전송"""
        try:
            response_data = json.dumps(data, ensure_ascii=False, indent=2)
            
            self.send_response(200)
            self.send_header('Content-Type', 'application/json; charset=utf-8')
            self.send_header('Access-Control-Allow-Origin', '*')
            self.send_header('Cache-Control', 'no-cache')
            self.end_headers()
            
            self.wfile.write(response_data.encode('utf-8'))
            
        except (BrokenPipeError, ConnectionResetError):
            # 클라이언트 연결이 끊어진 경우 조용히 처리
            logger.info("클라이언트 연결이 끊어짐")
        except Exception as e:
            logger.error(f"JSON 응답 전송 오류: {e}")
            try:
                self.send_error_response("응답 전송 중 오류가 발생했습니다")
            except (BrokenPipeError, ConnectionResetError):
                logger.info("오류 응답 전송 중 클라이언트 연결 끊어짐")
    
    def send_error_response(self, message, status=500):
        """오류 응답 전송"""
        try:
            error_data = {
                'error': True,
                'message': message,
                'timestamp': datetime.now().isoformat()
            }
            
            response_data = json.dumps(error_data, ensure_ascii=False, indent=2)
            
            self.send_response(status)
            self.send_header('Content-Type', 'application/json; charset=utf-8')
            self.send_header('Access-Control-Allow-Origin', '*')
            self.end_headers()
            
            self.wfile.write(response_data.encode('utf-8'))
        except (BrokenPipeError, ConnectionResetError):
            logger.info("오류 응답 전송 중 클라이언트 연결 끊어짐")

def run_server(port=5000):
    """서버 실행"""
    server_address = ('0.0.0.0', port)
    httpd = HTTPServer(server_address, YonginVentilationHandler)
    
    logger.info(f"교실 환기 판단 서버가 포트 {port}에서 실행 중입니다...")
    logger.info("실제 경기도 대기질 API 연동 중...")
    
    try:
        httpd.serve_forever()
    except KeyboardInterrupt:
        logger.info("서버 종료 중...")
        httpd.shutdown()

if __name__ == '__main__':
    run_server()
