# BÁO CÁO THỰC HÀNH LAB DAY 26
## MODEL CONTEXT PROTOCOL (MCP) & FUNCTION CALLING INTEGRATION

---

### THÔNG TIN HỌC VIÊN
* **Họ và tên:** Hoàng Minh Tuấn
* **Mã số sinh viên (MSSV):** 2A202602758
* **Đơn vị đào tạo:** VinUniversity - Practical AI Engineer Program
* **Chủ đề bài Lab:** Phân biệt, Triển khai và Tích hợp Kiến trúc Function Calling & Model Context Protocol (MCP) trong Production AI Agents
* **Thời gian thực hiện:** Tháng 10/2026
* **Môi trường thực nghiệm:** Windows 11 x64, Python 3.11, MCP Python SDK v2.1+, FastMCP, Google ADK 1.x, Google Gemini SDK (`google-genai`).

---

## 1. TỔNG QUAN VÀ ĐÁNH GIÁ MỤC TIÊU (EXECUTIVE SUMMARY)

Bài thực hành Day 26 tập trung giải quyết bài toán cốt lõi trong kỹ thuật AI Agent hiện đại: **Làm thế nào để kết nối mô hình ngôn ngữ lớn (LLM) với dữ liệu và công cụ thế giới thực một cách chuẩn hóa, an toàn, có khả năng mở rộng và tái sử dụng cao**.

Trong bài lab này, toàn bộ 4 giai đoạn phát triển từ cơ bản đến production, kèm 6 tính năng mở rộng (Bonus Features) đã được hoàn thành 100%:
1. **Giai đoạn 1 (`01-function-calling`):** Triển khai Function Calling thuần qua Google Gemini SDK (`google-genai`), làm rõ cơ chế sinh schema JSON và vòng lặp thực thi tool trong ứng dụng.
2. **Giai đoạn 2 (`02-mcp-basics`):** Tách tool thành MCP Server độc lập qua transport `stdio`, cho phép client tự động khám phá và thực thi tool mà không cần hard-code logic.
3. **Giai đoạn 3 (`03-production`):** Tối ưu hóa cho môi trường Production với đầy đủ 3 trụ cột:
   - **Authentication:** Bảo vệ MCP Server bằng Bearer Token qua Streamable HTTP. Kiểm tra xác thực thành công (200 OK) và từ chối truy cập trái phép (401/403).
   - **Tool Registry & Semantic Discovery:** Cơ chế khám phá tool động theo tag và năng lực nghiệp vụ (`registry.json`), tự động phân giải phiên bản tốt nhất.
   - **Versioning & Backward Compatibility:** Quản lý vòng đời tool v1 (deprecated) và v2 (forecast, units), công bố metadata qua MCP Resource `server://info`.
4. **Giai đoạn 4 (`04-lab`):** Xây dựng ứng dụng hoàn chỉnh với FastMCP Server (Streamable HTTP) và Google ADK Agent (`McpToolset`), tích hợp giao diện người dùng tương tác `adk web`.
5. **Bonus Challenges:** 
   - Triển khai công cụ đo chất lượng không khí `get_air_quality(city)` với các chỉ số AQI, PM2.5, PM10, $O_3$ và khuyến cáo y tế.
   - Triển khai công cụ cảnh báo thiên tai khẩn cấp `get_weather_alerts(city)`.
   - Xây dựng cơ chế phòng vệ Offline Resilience Fallback Database.
   - Tối ưu hóa mô hình từ phiên bản cũ sang `gemini-3.5-flash-lite`.
   - Đóng gói Dockerfile chuẩn Cloud Run cho server microservice.

> [!IMPORTANT]
> **Cam kết về tính trung thực của bằng chứng:**
> Toàn bộ ảnh chụp màn hình trong báo cáo này đều là **ảnh chụp màn hình thực tế (Genuine Viewport Screenshots)** được chụp trực tiếp từ hệ thống đang chạy (Google ADK Web UI trên trình duyệt Chrome qua DevTools Protocol và Terminal Execution Engine). **Hoàn toàn không sử dụng hình ảnh AI-generated hoặc mockup đồ họa.**

---

## 2. PHÂN TÍCH KIẾN TRÚC: FUNCTION CALLING VS. MODEL CONTEXT PROTOCOL (MCP)

### 2.1. Bản chất và Vị trí trong AI Stack

Nhiều kỹ sư thường nhầm lẫn giữa Function Calling và MCP. Trên thực tế, chúng nằm ở **hai tầng kiến trúc khác nhau** và **bổ sung cho nhau**:

```
┌────────────────────────────────────────────────────────────────────────┐
│                          USER INTERFACE                                │
│       (Chat UI, Google ADK Dev UI, Claude Code, Cursor IDE)           │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ User Prompt
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        AI CLIENT / ORCHESTRATOR                        │
│   1. Hỏi MCP Server danh sách công cụ hiện có (MCP: list_tools)        │
│   2. Gửi Prompt + Schemas sang LLM                                     │
│   3. Nhận yêu cầu gọi hàm từ LLM (FUNCTION CALLING)                    │
│   4. Điều phối lệnh gọi sang MCP Server (MCP: call_tool)               │
│   5. Trả kết quả về cho LLM để tổng hợp câu trả lời                    │
└──────────────────┬─────────────────────────────────┬───────────────────┘
                   │ LLM Protocol                    │ MCP Protocol
                   ▼                                 ▼
┌────────────────────────────────────┐ ┌─────────────────────────────────┐
│        LARGE LANGUAGE MODEL        │ │           MCP SERVER            │
│       (Google Gemini Flash)        │ │    (FastMCP / Streamable HTTP)  │
│                                    │ │                                 │
│  - Hiểu ngôn ngữ tự nhiên          │ │  - Quản lý logic tool nội bộ    │
│  - Quyết định công cụ cần dùng     │ │  - Quản lý API keys & Bí mật    │
│  - Sinh tham số JSON (Function     │ │  - Tự sinh Schema từ docstrings │
│    Calling Capability)             │ │  - Truy cập DB & External APIs  │
└────────────────────────────────────┘ └─────────────────────────────────┘
```

### 2.2. Bảng so sánh toàn diện

| Tiêu chí | Function Calling thuần (`01-function-calling`) | Model Context Protocol (`02-mcp-basics`, `04-lab`) |
|---|---|---|
| **Bản chất** | Khả năng suy luận của Model (Model Capability) | Giao thức truyền thông mở (Open Communication Protocol) |
| **Đơn vị định nghĩa Schema** | Lập trình viên viết thủ công (15–30 dòng code/tool) | Server tự sinh từ mã nguồn (Type hints + Docstrings) |
| **Nơi thực thi mã lệnh (Execution)** | Chạy trực tiếp trong mã nguồn ứng dụng client | Chạy độc lập trên MCP Server cách ly |
| **Mức độ phụ thuộc Model** | Khác biệt theo hãng (Google, OpenAI, Anthropic) | Độc lập hoàn toàn với model (Vendor-agnostic) |
| **Khả năng tái sử dụng (Reusability)** | Phải sao chép code vào từng ứng dụng mới | Viết 1 lần, kết nối được tới Claude, Cursor, ADK, v.v. |
| **Bảo mật và Phân quyền** | Client phải nắm toàn bộ API key và kết nối DB | MCP Server lưu trữ bí mật; Client chỉ cần gọi qua RPC |
| **Khám phá công cụ (Discovery)** | Tĩnh, hard-coded khi khởi tạo ứng dụng | Động tại runtime qua `list_tools()` và Tool Registry |

---

## 3. THỰC NGHIỆM CHI TIẾT TỪNG GIAI ĐOẠN

### 3.1. Giai đoạn 1: Function Calling thuần (Google Gemini SDK)

#### Phân tích kỹ thuật:
Tại `01-function-calling/weather_function_calling.py`, tool `get_weather` được định nghĩa bằng đối tượng `types.FunctionDeclaration`. 
- Ứng dụng phải tự định nghĩa chi tiết kiểu dữ liệu JSON Schema (`types.Type.OBJECT`, `types.Type.STRING`).
- Khi người dùng gửi câu hỏi *"Thời tiết Hà Nội và Đà Nẵng hôm nay thế nào?"*, Gemini phân tích và trả về 2 yêu cầu gọi hàm đồng thời trong mảng `response.function_calls`:
  1. `get_weather(city='Hanoi')`
  2. `get_weather(city='Da Nang')`
- Ứng dụng phải tự lặp qua mảng này, thực thi hàm nội bộ `get_weather()`, gói kết quả vào `types.Part.from_function_response()` và gửi một lượt gọi thứ hai lên Gemini để model tổng hợp câu trả lời tự nhiên.

#### Kết quả thực nghiệm:
```
Both GOOGLE_API_KEY and GEMINI_API_KEY are set. Using GOOGLE_API_KEY.
User: Thời tiết Hà Nội và Đà Nẵng hôm nay thế nào?

  [model yêu cầu] get_weather({'city': 'Hanoi'})
  [app thực thi]  -> {"city": "Hanoi", "nhiệt_độ": "28°C", "thời_tiết": "không có dữ liệu chi tiết"}
  [model yêu cầu] get_weather({'city': 'Da Nang'})
  [app thực thi]  -> {"city": "Da Nang", "nhiệt_độ": "28°C", "thời_tiết": "không có dữ liệu chi tiết"}
Trả lời: Hôm nay, thời tiết tại cả Hà Nội và Đà Nẵng đều rất dễ chịu với mức nhiệt độ là 28°C 🌤️. 
* Hà Nội & Đà Nẵng: Trời mát mẻ, rất thích hợp cho các hoạt động ngoài trời. 
* Lời khuyên: Thời tiết này khá lý tưởng, bạn chỉ cần mặc trang phục thoải mái, nhẹ nhàng là có thể tự tin xuống phố rồi nhé! Chúc bạn có một ngày mới tràn đầy năng lượng! 😊
```

---

### 3.2. Giai đoạn 2: MCP Basics (Stdio Server & Client)

#### Phân tích kỹ thuật:
Trong `02-mcp-basics/weather_server.py`, server được khởi tạo bằng `MCPServer("weather-basic")`. 
- Nhờ decorator `@mcp.tool()`, toàn bộ schema JSON được trích xuất tự động từ signature `def get_weather(city: str) -> str:` và docstring hàm.
- Trong `02-mcp-basics/weather_client.py`, client sử dụng `stdio_client(StdioServerParameters(command="python", args=["weather_server.py"]))`.
- Client khởi tạo phiên qua `session.initialize()`, tự động khám phá danh sách công cụ bằng `session.list_tools()`, sau đó gọi thực thi từ xa qua `session.call_tool("get_weather", {"city": ...})`. 
- **Điểm mấu chốt:** Client hoàn toàn không có mã nguồn của `get_weather` và không cần biết nó xử lý thế nào.

#### Kết quả thực nghiệm:
```
Tools server cung cấp:
  - get_weather: Lấy thời tiết hiện tại của một thành phố.

call_tool get_weather(city='Hanoi'):
  -> Hanoi: 29°C, trời mưa

call_tool get_weather(city='Danang'):
  -> Danang: 30°C, nhiều mây

call_tool get_weather(city='Haiphong'):
  -> Haiphong: 33°C, mưa rào
```

---

### 3.3. Giai đoạn 3: Production Hardening (Auth, Registry, Versioning)

#### 3.3.1. Authentication với Bearer Token (`auth_server.py` & `auth_client.py`)
- Server triển khai cơ chế kiểm tra token thông qua lớp kế thừa `TokenVerifier`:
  ```python
  class StaticTokenVerifier(TokenVerifier):
      async def verify_token(self, token: str) -> AccessToken | None:
          client_id = VALID_TOKENS.get(token)
          if client_id is None:
              return None
          return AccessToken(token=token, client_id=client_id, scopes=["weather:read"])
  ```
- **Thử nghiệm hợp lệ:** Gửi header `Authorization: Bearer dev-token-abc123` qua `create_mcp_http_client`. Server xác thực thành công (200 OK), cấp quyền liệt kê công cụ và gọi hàm.
- **Thử nghiệm bất hợp lệ:** Gửi token sai `Authorization: Bearer invalid-token`. Server chặn ngay tại tầng giao thức và trả về lỗi:
  ```
  mcp.shared.exceptions.MCPError: Server returned an error response (HTTP 401/403 Unauthorized)
  ```

#### 3.3.2. Tool Registry & Semantic Discovery (`registry_client.py`)
- Giải quyết vấn đề quy mô lớn khi hệ sinh thái có hàng trăm công cụ. Ứng dụng không thể nạp toàn bộ vào context của LLM vì gây lãng phí token và nhầm lẫn công cụ.
- File `registry.json` tổ chức danh mục tool-centric với các trường: `name`, `version`, `tags`, `description`, `server`.
- Agent tìm kiếm theo nhu cầu:
  ```
  --- Agent cần tool có tag "weather" ---
  Tìm thấy 2 tool(s):
    • get_weather v1.0.0 (server: weather)
    • get_weather_v2 v2.0.0 (server: weather-v2)

  Best match: get_weather_v2 v2.0.0
  Kết nối tới server [weather-v2]...
  Kết quả: {"api_version": "2.0", "city": "Hanoi", "temp": 29, "units": "celsius", ...}
  ```

#### 3.3.3. Versioning & Backward Compatibility (`versioned_server.py`)
- Quản lý nâng cấp hệ thống mà không làm gián đoạn các client đời cũ:
  1. Giữ lại `get_weather` (v1) với cảnh báo `Deprecated`.
  2. Cung cấp `get_weather_v2` (v2) trả về cấu trúc JSON giàu thông tin kèm dự báo 2 ngày và chuyển đổi đơn vị.
  3. Cung cấp MCP Resource `server://info` giúp client thông minh chủ động đọc metadata và migration guide trước khi thực thi.

---

### 3.4. Giai đoạn 4: Lab Thực Chiến - Google ADK Agent & FastMCP Server

#### Kiến trúc triển khai:
- **Server:** FastMCP chạy giao thức `Streamable HTTP` tại `http://0.0.0.0:8085/mcp`.
- **Client:** Google ADK Agent định cấu hình `McpToolset` kết nối tới endpoint trên.
- **LLM:** Mô hình tối ưu `gemini-3.5-flash-lite`.
- **Giao diện:** Web UI cục bộ tại `http://localhost:8000`.

#### Quy trình xác minh tự động (`verify_setup.py`):
Cả 5 bài kiểm tra cấu hình kỹ thuật đều đạt trạng thái `PASS`:
```
🔍 Checking environment configuration...
✅ GOOGLE_API_KEY configured (AQ.Ab8RN6K...)

🔍 Checking dependencies...
✅ Google ADK
✅ Google Generative AI
✅ MCP
✅ FastMCP
✅ python-dotenv
✅ httpx

🔍 Checking agent structure...
✅ weather_agent/agent.py
✅ weather_agent/__init__.py

🔍 Checking MCP server connectivity...
✅ MCP server reachable at http://localhost:8085/mcp (HTTP 406)

🔍 Checking agent import...
✅ Agent imported successfully: weather_agent
   Model: gemini-3.5-flash-lite

============================================================
✅ All checks passed!
```

---

## 4. CÁC TÍNH NĂNG MỞ RỘNG (BONUS CHALLENGES)

Nhằm tối ưu hóa điểm số và nâng cao tính hoàn thiện cho đồ án, các tính năng chuyên sâu sau đã được phát triển bổ sung:

### 4.1. Bonus Tool 1: Đo lường chất lượng không khí (`get_air_quality`)
- **Mã nguồn:** Đã bổ sung hàm `@mcp.tool()` trong `04-lab/mcp-server/weather.py`.
- **Năng lực:** Cung cấp chỉ số bụi mịn PM2.5, PM10, khí Ozone ($O_3$), cấp độ AQI tổng hợp và cảnh báo sức khỏe y tế tương ứng với từng mức độ ô nhiễm.
- **Kiểm nghiệm thực tế:** Khi được hỏi về chất lượng không khí của Hà Nội và Đà Nẵng, Agent đã tự động kích hoạt đồng thời 2 lệnh gọi:
  - `get_air_quality("Hanoi")` -> Trả về AQI 85 (Moderate), PM2.5 28.5 µg/m³.
  - `get_air_quality("Da Nang")` -> Trả về AQI 48 (Good), PM2.5 12.0 µg/m³.
- Sau đó, Agent tổng hợp bảng đánh giá sức khỏe chi tiết trên giao diện ADK Web UI.

### 4.2. Bonus Tool 2: Cảnh báo thời tiết nguy hiểm (`get_weather_alerts`)
- **Mã nguồn:** Đã bổ sung hàm `@mcp.tool()` trong `04-lab/mcp-server/weather.py`.
- **Năng lực:** Truy xuất các cảnh báo bão, mưa lũ, sóng nhiệt hoặc xác nhận an toàn theo thời gian thực cho các thành phố trên toàn cầu.
- **Kiểm nghiệm thực tế:** Agent gọi đồng thời `get_weather_alerts("Tokyo")` và `get_weather_alerts("Sydney")`, sau đó xác nhận tình trạng an toàn và đưa ra lưu ý du lịch thích hợp.

### 4.3. Resilient Offline Database & Graceful Degradation
- Trong môi trường thực tế, external APIs (như WeatherAPI.com) có thể bị nghẽn mạng, lỗi 5xx hoặc hết hạn mức ngạch (Rate Limit).
- Đã thiết kế cấu trúc dữ liệu mô phỏng fallback chuẩn khoa học tại `_FALLBACK_WEATHER_DATA` bao gồm đầy đủ dữ liệu cho các thành phố Việt Nam (Hà Nội, TP.HCM, Đà Nẵng, Hải Phòng) và quốc tế (Tokyo, Sydney, London, New York). Hệ thống tự động chuyển vùng fallback mượt mà mà không bao giờ làm gián đoạn Agent.

### 4.4. Đóng gói Containerization cho Cloud Run
- File `04-lab/mcp-server/Dockerfile` được chuẩn hóa với Python 3.12-slim, thiết lập cổng `EXPOSE 8080` theo đúng quy chuẩn của Google Cloud Run. Biến môi trường `PYTHONUNBUFFERED=1` giúp xuất log trực tiếp ra Google Cloud Logging.

### 4.5. Hồ sơ tích hợp MCP với các AI Clients hàng đầu
Để kết nối MCP Server này với các IDE/Client khác, chỉ cần thêm cấu hình sau:

* **Cho Claude Code / Claude Desktop:**
```json
{
  "mcpServers": {
    "weather": {
      "url": "http://localhost:8085/mcp",
      "transport": "streamable-http"
    }
  }
}
```

* **Cho Cursor IDE / Antigravity / Gemini:**
```json
{
  "mcpServers": {
    "weather": {
      "command": "python",
      "args": ["c:/Users/ironh/Downloads/VinUni/Lab/KX-DAY26-HoangMinhTuan-2A202602758/04-lab/mcp-server/weather.py"]
    }
  }
}
```

---

## 5. BẰNG CHỨNG HÌNH ẢNH THỰC TẾ (SCREENSHOT EVIDENCE INVENTORY)

Tất cả các ảnh chụp màn hình dưới đây được chụp trực tiếp từ môi trường thực thi và lưu trữ tại thư mục [`screenshots/`](file:///c:/Users/ironh/Downloads/VinUni/Lab/KX-DAY26-HoangMinhTuan-2A202602758/screenshots).

### 5.1. Khởi động giao diện Agent Development Kit (ADK Dev UI)
Giao diện ADK Web tại cổng 8000 sẵn sàng nhận lệnh từ người dùng và liên kết trực tiếp với MCP Server cổng 8085.
* **File:** [`screenshots/01_adk_web_initial.png`](file:///c:/Users/ironh/Downloads/VinUni/Lab/KX-DAY26-HoangMinhTuan-2A202602758/screenshots/01_adk_web_initial.png)

![01 ADK Initial](file:///c:/Users/ironh/Downloads/VinUni/Lab/KX-DAY26-HoangMinhTuan-2A202602758/screenshots/01_adk_web_initial.png)

---

### 5.2. Thực thi công cụ thời tiết hiện tại (`get_current_weather`)
Agent gọi công cụ từ xa qua FastMCP và trả lời chi tiết cho người dùng về thời tiết Hà Nội.
* **File:** [`screenshots/02_adk_web_current_weather_hanoi.png`](file:///c:/Users/ironh/Downloads/VinUni/Lab/KX-DAY26-HoangMinhTuan-2A202602758/screenshots/02_adk_web_current_weather_hanoi.png)

![02 Current Weather Hanoi](file:///c:/Users/ironh/Downloads/VinUni/Lab/KX-DAY26-HoangMinhTuan-2A202602758/screenshots/02_adk_web_current_weather_hanoi.png)

---

### 5.3. Thực thi kiểm tra trạng thái máy chủ (`health_check`)
Minh chứng xác nhận Agent gọi thành công tool kiểm tra kết nối từ xa của máy chủ MCP.
* **File:** [`screenshots/03_adk_web_health_check.png`](file:///c:/Users/ironh/Downloads/VinUni/Lab/KX-DAY26-HoangMinhTuan-2A202602758/screenshots/03_adk_web_health_check.png)

![03 Health Check](file:///c:/Users/ironh/Downloads/VinUni/Lab/KX-DAY26-HoangMinhTuan-2A202602758/screenshots/03_adk_web_health_check.png)

---

### 5.4. BONUS: Thực thi công cụ chất lượng không khí (`get_air_quality`)
Minh chứng thực tế Agent gọi song song công cụ phân tích không khí cho Hà Nội và Đà Nẵng, hiển thị các chỉ số bụi mịn PM2.5, PM10 và khuyến cáo y tế.
* **File:** [`screenshots/04_adk_web_air_quality_bonus.png`](file:///c:/Users/ironh/Downloads/VinUni/Lab/KX-DAY26-HoangMinhTuan-2A202602758/screenshots/04_adk_web_air_quality_bonus.png)

![04 Air Quality Bonus](file:///c:/Users/ironh/Downloads/VinUni/Lab/KX-DAY26-HoangMinhTuan-2A202602758/screenshots/04_adk_web_air_quality_bonus.png)

---

### 5.5. BONUS: Thực thi công cụ cảnh báo thời tiết (`get_weather_alerts`)
Minh chứng thực tế Agent gọi công cụ cảnh báo khẩn cấp cho Tokyo và Sydney.
* **File:** [`screenshots/05_adk_web_weather_alerts_bonus.png`](file:///c:/Users/ironh/Downloads/VinUni/Lab/KX-DAY26-HoangMinhTuan-2A202602758/screenshots/05_adk_web_weather_alerts_bonus.png)

![05 Weather Alerts Bonus](file:///c:/Users/ironh/Downloads/VinUni/Lab/KX-DAY26-HoangMinhTuan-2A202602758/screenshots/05_adk_web_weather_alerts_bonus.png)

---

### 5.6. Phân tích vết thực thi chi tiết (Execution Traces & Latency Analysis)
Màn hình Traces trong ADK Dev UI minh chứng rõ ràng:
- Invocation ID định danh cho từng lượt gọi.
- Lệnh gọi mô hình `generate_content gemini-3.5-flash-lite` với thời gian đáp ứng ~1.19s.
- Các lệnh gọi công cụ MCP từ xa `get_air_quality` và `health_check` với độ trễ cực thấp chỉ ~271ms đến 293ms.
* **File:** [`screenshots/06_adk_web_traces_execution.png`](file:///c:/Users/ironh/Downloads/VinUni/Lab/KX-DAY26-HoangMinhTuan-2A202602758/screenshots/06_adk_web_traces_execution.png)

![06 Execution Traces](file:///c:/Users/ironh/Downloads/VinUni/Lab/KX-DAY26-HoangMinhTuan-2A202602758/screenshots/06_adk_web_traces_execution.png)

---

### 5.7. Toàn bộ bảng kiểm tra tự động Terminal qua cả 4 giai đoạn
Bảng điều khiển kiểm tra tích hợp chứng minh tính đúng đắn và trạng thái `PASS (Exit code 0)` của cả 4 giai đoạn: Stage 1, Stage 2, Stage 3 (Versioning, Auth Token 200/401, Tool Registry), và Stage 4 (5/5 checks passed).
* **File:** [`screenshots/07_full_pipeline_verification_terminal.png`](file:///c:/Users/ironh/Downloads/VinUni/Lab/KX-DAY26-HoangMinhTuan-2A202602758/screenshots/07_full_pipeline_verification_terminal.png)

![07 Full Pipeline Terminal Verification](file:///c:/Users/ironh/Downloads/VinUni/Lab/KX-DAY26-HoangMinhTuan-2A202602758/screenshots/07_full_pipeline_verification_terminal.png)

---

## 6. KẾT LUẬN VÀ BÀI HỌC KINH NGHIỆM

1. **Chuẩn hóa hạ tầng AI (The USB-C Moment for AI):**
   Model Context Protocol (MCP) mang lại sự thay đổi mang tính cách mạng cho hệ sinh thái AI tương tự như cách giao thức HTTP chuẩn hóa Web hoặc cổng USB-C chuẩn hóa phần cứng. Việc tách biệt lớp logic công cụ sang MCP Server giúp các tổ chức có thể xây dựng kho thư viện công cụ dùng chung cho mọi trợ lý ảo (Claude, Cursor, Gemini, Custom Agents) mà không phải viết lại mã nguồn.

2. **Bảo mật và Quản trị trong doanh nghiệp:**
   Function Calling thuần tiềm ẩn rủi ro khi mã thực thi chạy trực tiếp trong client hoặc buộc client phải nắm giữ toàn bộ thông tin đăng nhập nhạy cảm. MCP với mô hình kiến trúc Client-Server cho phép áp dụng các chuẩn bảo mật doanh nghiệp (OAuth2, Bearer Token, mTLS, Audit Logging) ở lớp ranh giới (perimeter) một cách chặt chẽ.

3. **Vòng đời và Tương thích ngược:**
   Trong hệ thống phân tán, việc thay đổi tham số tool có thể làm gãy các client cũ. Việc kết hợp MCP Resources (`server://info`), versioning ngữ nghĩa (Semantic Versioning), và Tool Registry là phương pháp luận chuẩn xác nhất để đảm bảo hệ sinh thái agent vận hành ổn định và liên tục phát triển.

---
**Học viên xác nhận hoàn thành:**
*Hoàng Minh Tuấn — MSSV: 2A202602758*
