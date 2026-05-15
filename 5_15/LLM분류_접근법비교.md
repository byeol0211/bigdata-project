# %% [1] HTTP 텍스트 재구성 + 프롬프트 함수
def build_http_text(row) -> str:
    method = row.get("method", "GET")
    url    = unquote(str(row.get("url", "")), encoding="latin-1")
    body   = str(row.get("body_decoded", row.get("body", "")) or "")
    text   = f"{method} {url} HTTP/1.1"
    if body and body != "nan":
        text += f"\nBody: {body[:200]}"
    return text


PROMPT_TEMPLATE = 'You are a web security expert. Classify each HTTP request as "Normal" or "Anomalous" and provide a brief reason.\n\nExamples:\nRequest: GET /index.jsp HTTP/1.1\nOutput: {{"label": "Normal", "reason": "Standard page request, no suspicious pattern"}}\n\nRequest: GET /search?q=\' OR \'1\'=\'1 HTTP/1.1\nOutput: {{"label": "Anomalous", "reason": "Classic SQL Injection pattern with OR 1=1"}}\n\nNow classify:\nRequest: {http_text}\nOutput:'


def classify_with_llm(http_text: str, model: str = "gemma3:4b") -> dict:
    """Ollama로 HTTP 요청 분류 -> {label, reason}"""
    prompt = PROMPT_TEMPLATE.format(http_text=http_text)
    response = ollama.chat(
        model=model,
        messages=[{"role":"user","content":prompt}],
        options={"temperature": 0},  # 결정성 높이기
    )
    text = response["message"]["content"]

    # JSON 추출 - LLM이 가끔 앞뒤 설명을 붙임
    match = re.search(r"\{[^{}]*\}", text, re.DOTALL)
    if not match:
        return {"label":"Unknown", "reason": text[:80]}
    try:
        return json.loads(match.group())
    except json.JSONDecodeError:
        return {"label":"Unknown", "reason": text[:80]}


# 단건 테스트
test_text = "GET /tienda1/publico/anadir.jsp?id=2'+OR+'1'='1 HTTP/1.1"
print("입력:", test_text)
print("응답:", classify_with_llm(test_text))

입력: GET /tienda1/publico/anadir.jsp?id=2'+OR+'1'='1 HTTP/1.1
응답: {'label': 'Anomalous', 'reason': "SQL Injection attempt. The request includes ' OR '1'='1', a common SQL injection pattern designed to bypass authentication or retrieve all data."}

# %% [3] 정확도/F1 계산
llm_df["pred_clean"] = llm_df["pred"].replace({"Unknown":"Normal"})
y_true = (llm_df["true"] == "Anomalous").astype(int)
y_pred = (llm_df["pred_clean"] == "Anomalous").astype(int)

llm_acc = accuracy_score(y_true, y_pred)
llm_f1  = f1_score(y_true, y_pred)

print(f"LLM 정확도: {llm_acc:.4f}")
print(f"LLM F1:    {llm_f1:.4f}")
print(f"분류 실패(Unknown): {(llm_df['pred']=='Unknown').sum()}건")
print()
print(classification_report(y_true, y_pred, target_names=["Normal","Anomalous"]))

LLM 정확도: 0.8400
LLM F1:    0.8519
분류 실패(Unknown): 1건

              precision    recall  f1-score   support

      Normal       0.84      0.81      0.83        47
   Anomalous       0.84      0.87      0.85        53

    accuracy                           0.84       100
   macro avg       0.84      0.84      0.84       100
weighted avg       0.84      0.84      0.84       100

---------------------------------------------------------------------------------------------------------------------------------------

# %% [1] HTTP 텍스트 재구성 + 프롬프트 함수
def build_http_text(row) -> str:
    method = row.get("method", "GET")
    url    = unquote(str(row.get("url", "")), encoding="latin-1")
    body   = str(row.get("body_decoded", row.get("body", "")) or "")
    text   = f"{method} {url} HTTP/1.1"
    if body and body != "nan":
        text += f"\nBody: {body[:200]}"
    return text

# 프롬프트 전략: [역할] - [위험군 정의] - [제약사항]
PROMPT_TEMPLATE = """You are a web security expert. 
Classify each HTTP request as "Normal" or "Anomalous" and provide a brief reason.

Examples:
Request: GET /index.jsp HTTP/1.1
Output: {{"label": "Normal", "reason": "Standard page request"}}

Request: GET /search?q=' OR '1'='1 HTTP/1.1
Output: {{"label": "Anomalous", "reason": "Classic SQL Injection"}}

Now classify:
Request: {http_text}
Output:"""

def classify_with_llm(http_text: str, model: str = "gemma3:4b") -> dict:
    """Ollama로 HTTP 요청 분류 -> {label, reason}"""
    prompt = PROMPT_TEMPLATE.format(http_text=http_text)
    response = ollama.chat(
        model=model,
        messages=[{"role":"user","content":prompt}],
        options={"temperature": 0},  # 결정성 높이기
    )
    text = response["message"]["content"]

    # JSON 추출 - LLM이 가끔 앞뒤 설명을 붙임
    match = re.search(r"\{[^{}]*\}", text, re.DOTALL)
    if not match:
        return {"label":"Unknown", "reason": text[:80]}
    try:
        return json.loads(match.group())
    except json.JSONDecodeError:
        return {"label":"Unknown", "reason": text[:80]}


# 단건 테스트
test_text = "GET /tienda1/publico/anadir.jsp?id=2'+OR+'1'='1 HTTP/1.1"
print("입력:", test_text)
print("응답:", classify_with_llm(test_text))
# %% [3] 정확도/F1 계산
llm_df["pred_clean"] = llm_df["pred"].replace({"Unknown":"Normal"})
y_true = (llm_df["true"] == "Anomalous").astype(int)
y_pred = (llm_df["pred_clean"] == "Anomalous").astype(int)

llm_acc = accuracy_score(y_true, y_pred)
llm_f1  = f1_score(y_true, y_pred)

print(f"LLM 정확도: {llm_acc:.4f}")
print(f"LLM F1:    {llm_f1:.4f}")
print(f"분류 실패(Unknown): {(llm_df['pred']=='Unknown').sum()}건")
print()
print(classification_report(y_true, y_pred, target_names=["Normal","Anomalous"]))

LLM 정확도: 0.8500
LLM F1:    0.8673
분류 실패(Unknown): 1건

              precision    recall  f1-score   support

      Normal       0.90      0.77      0.83        47
   Anomalous       0.82      0.92      0.87        53

    accuracy                           0.85       100
   macro avg       0.86      0.85      0.85       100
weighted avg       0.86      0.85      0.85       100