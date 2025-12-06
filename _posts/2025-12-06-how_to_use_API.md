---
layout: post
title: "使用API进行LLM-as-a-judge及API相关指令（以Gemini为例）"
date: 2025-12-06
tags: [科研日记]
toc: true
author: WqW-hit
---

今天这份博客主要以Google为例介绍如何使用API完成推理以及一些常见的API操作指令，让你对API内部的一些信息有更好的认识。

## 1. 背景

最近接到了一项任务，需要调用Gemini对模型的CoT进行评估&分析。所以接下来简单介绍一下**LLM-as-a-judge技术**以及一些使用方法。

在LLM-as-a-judge技术出现之前，为了衡量模型的长文本回答（如分析模型对一段文本/视频的总结分析是否到位），我们一般会使用两种方法，要么找真人给回答打分，要么用类似于BLEU/ROUGE这种比较字符串相似度的硬性指标直接计算。但这两种方法的缺点都很明显：前者太吃经济实力，对现在动辄几千条几万条甚至更多的数据集，如果需要完全用真人**要花太多的钱和时间**，同时还有培训成本（当然如果你是黑心奴隶主主打一个压迫那当我没说）；后者**太笨**，在这种指标评价下，“开心”和“高兴”完全不一样，可能会被打0分，但事实上这两个词在语义上是几乎一致的。

在这样的背景之下，LLM-as-a-judge技术应运而生。简单来说就是**请一个高级AI（如Gemini3，Claude4.5 Sonnet和GPT 5等）来给其他低级AI的作业打分**。低级AI在一个benchmark上做出回答之后将原数据以及低级AI的回答一并输入高级AI，并给出类似“你是一个公正的老师。请对比选手的回答和正确答案。如果意思一致，请打 1 分，否则打 0 分。请解释你的理由。”的指令，让高级AI对低级AI的回答进行打分和评价。

这种技术在人工阅卷和硬性指标阅卷之间找到了一个平衡点，既能够较快&自动化的实现评测，也能够做出和人类评价相当的回答。但是这种技术不是完全没有缺点的， 比如裁判 AI 可能会因为喜欢某种说话风格（比如喜欢长篇大论）而给高分，哪怕答案其实是在胡言乱语；或者只有当裁判AI的水平高于选手AI的时候才能够相对较好的打分。

## 2. 具体方法

简单介绍完了LLM-as-a-judge技术，接下来就是使用方法以及操作过程中的一些注意事项。

下面就是我自己使用的调用Gemini API对模型的CoT进行打分的代码，我将结合代码内的一些细节进行解释

```python
import os
import json
import time
from pydantic import BaseModel, Field
from tqdm import tqdm
import google.generativeai as genai
import google.generativeai.types as genai_types

# API Keys 列表
API_KEYS = [
    "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
]

def get_configured_client(api_key):
    try:
        genai.configure(api_key=api_key)
        return genai.GenerativeModel(
            model_name="gemini-2.5-flash",
            # model_name="gemini-2.5-pro",
        )
    except Exception as e:
        print(f"Error configuring Gemini client with key {api_key[:10]}...: {e}")
        return None

# --- 2. 定义评估维度和输出格式 ---

class EvaluationResult(BaseModel):
    """
    定义了LLM-as-a-judge的输出结构，用于评估模型CoT的质量。
    """
    reasoning_correctness_score: int = Field(
        ..., ge=1, le=5, description="分数 1-5: 推理过程是否正确反映了视频内容，逻辑是否无误。"
    )
    reasoning_consistency_score: int = Field(
        ..., ge=1, le=5, description="分数 1-5: 推理过程是否与最终答案保持一致，无内部矛盾。"
    )
    reasoning_redundancy_score: int = Field(
        ..., ge=1, le=5, description="分数 1-5: 推理过程是否简洁，有无不必要的重复。5分代表最简洁。"
    )
    final_answer_correctness: bool = Field(
        ..., description="最终答案是否与'Ground Truth Answer'匹配 (True/False)。"
    )
    evaluation_rationale: str = Field(
        ..., description="对所有评分的详细解释，指出被评估模型回答的优点和缺点。"
    )

# --- 3.1 辅助函数 ---
def process_video_path(video_path):
    """
    处理视频路径，确保路径是绝对路径或者相对于当前工作目录的正确路径。
    这里可以根据实际情况添加更多处理逻辑，例如替换路径前缀等。
    """
    # 示例：如果视频路径是相对路径，可以尝试拼接一个基础路径
    # base_path = "/data/videos"
    # if not os.path.isabs(video_path):
    #     return os.path.join(base_path, video_path)
    return video_path

# --- 3. 设计评估Prompt模板 ---

EVALUATION_PROMPT_TEMPLATE = """
You are an impartial and meticulous AI evaluator. Your task is to critically assess the quality of a model's response to a video-based question. The model's response includes its step-by-step thinking process (Chain-of-Thought) and a final answer.

**You are provided with:**
1.  **The Video:** The primary source of information.
2.  **The Context and Question:** The user's query and any associated text or options.
3.  **The Ground Truth Answer:** The officially correct answer.
4.  **The Model's Full Response to be Evaluated:** This contains the model's entire reasoning process and its final conclusion.

**Your Evaluation Criteria:**

*   **1. Reasoning Correctness (Score 1-5):** Evaluate if the reasoning steps accurately interpret the video. Are the facts derived from the video correct? Is the logic sound?
    - 5: Perfectly accurate and logical.
    - 3: Contains minor errors or inaccuracies but the overall direction is correct.
    - 1: Completely hallucinatory or illogical.

*   **2. Reasoning Consistency (Score 1-5):** Evaluate if the final answer is a logical conclusion of the reasoning steps.
    - 5: The answer is perfectly supported by the reasoning.
    - 3: The answer is plausible but not strongly supported by the reasoning.
    - 1: The answer contradicts the reasoning.

*   **3. Reasoning Redundancy (Score 1-5):** Evaluate the conciseness of the reasoning.
    - 5: Highly concise and to the point. No redundant steps.
    - 3: Contains some repetition or slightly verbose steps.
    - 1: Extremely repetitive and convoluted.

*   **4. Final Answer Correctness (True/False):** Strictly compare the model's final answer with the ground truth answer.

**The Task to be Evaluated:**
---
**Context and Question:**
{question_text}

**Ground Truth Answer:**
{ground_truth_answer}

**Model's Full Response (CoT and Final Answer):**
{model_answer_cot}
---

Your output **MUST** be a single, valid JSON object that adheres to the following structure. Do not include any text, explanations, or markdown formatting outside of the JSON object.

Example JSON structure:
{{
  "reasoning_correctness_score": 5,
  "reasoning_consistency_score": 4,
  "reasoning_redundancy_score": 3,
  "final_answer_correctness": true,
  "evaluation_rationale": "The model correctly identified the key visual elements... However, the reasoning in step 3 was a repeat of step 2... The final answer is correct and consistent with the valid parts of the reasoning."
}}
"""

# --- 4. 主逻辑 ---

def main():
    # 输入：包含模型输出的JSON文件路径
    input_model_outputs_file = "xxxxxxxxxx"
    # 输出：保存评估结果的目录
    save_dir = "xxxxxxxxxxxxxxxxxxxxx"
    os.makedirs(save_dir, exist_ok=True)

    try:
        with open(input_model_outputs_file, "r") as f:
            dataset = json.load(f)
        if isinstance(dataset, dict): # 如果根是字典，则转换为项列表
             dataset = list(dataset.values())
    except (FileNotFoundError, json.JSONDecodeError) as e:
        print(f"Error loading dataset: {e}")
        return

    last_video_path = ""
    last_upload_api_key = None
    video_file_object = None

    for idx, item in enumerate(tqdm(dataset, desc="Evaluating Model Outputs")):
        question_id = item.get("question_id", f"item_{idx}")

        # 修复question_id包含非法字符（如/）的问题，因为它是文件名的一部分
        safe_question_id = question_id.replace("/", "_").replace("\\", "_")

        video_path = item.get("video_path")

        save_path = os.path.join(save_dir, f"{safe_question_id}_evaluation.json")
        # 检查是否已经评估过，如果文件存在且非空，则跳过
        if os.path.exists(save_path):
            try:
                with open(save_path, "r") as f:
                    # 尝试读取并解析JSON，如果成功且包含必要字段，则视为有效
                    existing_data = json.load(f)
                    if "llm_judge_evaluation" in existing_data:
                        print(f"Skipping {question_id}, already evaluated.")
                        continue
            except (json.JSONDecodeError, ValueError):
                print(f"Found corrupted or empty file for {question_id}, re-evaluating...")

        # 检查是否在之前的失败日志中（可选，但主要依靠文件是否存在来判断）
        # 即使在failed_items.log中，只要save_path不存在或无效，都会重新评估

        if not video_path:
            print(f"Skipping {question_id}, no video path provided.")
            continue

        # 处理视频路径
        processed_video_path = process_video_path(video_path)

        if not os.path.exists(processed_video_path):
            print(f"Video not found for {question_id}: {processed_video_path}")
            continue

        # 准备评估Prompt
        full_prompt = EVALUATION_PROMPT_TEMPLATE.format(
            question_text=item["question"],
            ground_truth_answer=item["answer"],
            model_answer_cot=item["model_answer"]
        )

        # 调用API进行评估 (整合上传逻辑以解决跨API文件访问问题)
        for attempt_idx, api_key in enumerate(API_KEYS):
            client = get_configured_client(api_key)
            if not client:
                continue

            # 检查是否需要上传/重新上传视频
            # 如果视频路径变了，或者当前API Key与上次上传的Key不一致，则必须重新上传
            if processed_video_path != last_video_path or api_key != last_upload_api_key or video_file_object is None:
                try:
                    print(f"\nUploading video: {processed_video_path} with API Key {attempt_idx + 1}")
                    # genai.configure 已在 get_configured_client 中调用，设置了全局 active client

                    video_file_object = genai.upload_file(path=processed_video_path)

                    # 等待视频处理完成
                    print(f"Waiting for video processing: {video_file_object.name}")
                    while video_file_object.state.name == "PROCESSING":
                        time.sleep(5)
                        video_file_object = genai.get_file(video_file_object.name)
                        print(f"Current state: {video_file_object.state.name}")

                    if video_file_object.state.name == "FAILED":
                        print(f"Video processing failed: {video_file_object.name}")
                        continue # 当前Key上传失败，尝试下一个Key

                    # 上传成功，更新状态
                    last_video_path = processed_video_path
                    last_upload_api_key = api_key
                except Exception as e:
                    print(f"Error uploading video with Key {attempt_idx + 1}: {e}")
                    continue # 当前Key上传出错，尝试下一个Key
            else:
                print(f"\nUsing previously uploaded video for {question_id} (Key {attempt_idx + 1}).")

            try:
                print(f"Generating evaluation for {question_id} with API Key {attempt_idx + 1}/{len(API_KEYS)}...")

                # 定义安全设置，禁用所有过滤以避免 PROHIBITED_CONTENT
                safety_settings = [
                    {
                        "category": "HARM_CATEGORY_HARASSMENT",
                        "threshold": "BLOCK_NONE"
                    },
                    {
                        "category": "HARM_CATEGORY_HATE_SPEECH",
                        "threshold": "BLOCK_NONE"
                    },
                    {
                        "category": "HARM_CATEGORY_SEXUALLY_EXPLICIT",
                        "threshold": "BLOCK_NONE"
                    },
                    {
                        "category": "HARM_CATEGORY_DANGEROUS_CONTENT",
                        "threshold": "BLOCK_NONE"
                    },
                ]

                response = client.generate_content(
                    contents=[full_prompt, video_file_object], # Prompt和视频文件
                    generation_config={
                        'response_mime_type': 'application/json',
                    },
                    safety_settings=safety_settings,
                    request_options={'timeout': 120} # 设置120秒超时
                )

                # 解析返回的JSON结果
                evaluation_data = json.loads(response.text)

                # 验证结果是否符合Pydantic模型 (可选但推荐)
                EvaluationResult(**evaluation_data) 

                # 准备最终保存文件
                final_output = {
                    "source_data": item,
                    "llm_judge_evaluation": evaluation_data
                }

                # 保存评估结果
                with open(save_path, "w", encoding="utf-8") as f:
                    json.dump(final_output, f, indent=2, ensure_ascii=False)

                print(f"Successfully saved evaluation for {question_id}")
                break # 成功后跳出重试循环

            except Exception as e:
                print(f"Error with API Key {attempt_idx + 1}: {e}")
                if attempt_idx == len(API_KEYS) - 1:
                    print(f"All API keys failed for {question_id}.")
                    # 记录失败的项，以便后续重试
                    with open(os.path.join(save_dir, "failed_items.log"), "a") as log_file:
                        log_file.write(f"{question_id}: {e}\n")
                else:
                    print("Switching to next API key...")
                    time.sleep(10) # 稍作等待再尝试下一个Key

        # API速率控制
        time.sleep(10) # 每次成功请求后稍作休息

if __name__ == "__main__":
    main()
```

1. 在代码内可以看到一个名为API_KEYS的列表，这里就是我们实际使用的模型的API，由于我要处理的数据集规模较大，单个API频繁调用容易被ban，所以我采用了API_KEYS列表的方式，当一个API出现异常时可以换另外一个API继续尝试
2. 在调用API的时候，如果中途出现了一些异常，可以使用下面的语句进行查询，只需要将`$API_KEY`换为你自己实际用的API_KEY以及将`<model_name>`改为你使用的model类型即可。

```bash
curl "https://generativelanguage.googleapis.com/v1beta/models/<model_name>?key=$API_KEY"
```

3. 当然，很多时候我们还会面对一个更加基础的问题：如何得到model_name？运行下面的语句即可，运行完之后你将会得到一个json列表，在列表的name字段中寻找即可

```bash
curl "https://generativelanguage.googleapis.com/v1beta/models?key=$API_KEY"
```

```json
{
  "models": [
    {
      "name": "models/gemini-2.5-flash",
      "version": "001",
      "displayName": "Gemini 2.5 Flash",
      "description": "Stable version of Gemini 2.5 Flash, our mid-size multimodal model that supports up to 1 million tokens, released in June of 2025.",
      "inputTokenLimit": 1048576,
      "outputTokenLimit": 65536,
      "supportedGenerationMethods": [
        "generateContent",
        "countTokens",
        "createCachedContent",
        "batchGenerateContent"
      ],
      "temperature": 1,
      "topP": 0.95,
      "topK": 64,
      "maxTemperature": 2,
      "thinking": true
    },
    {
      "name": "models/gemini-2.5-pro",
      "version": "2.5",
      "displayName": "Gemini 2.5 Pro",
      "description": "Stable release (June 17th, 2025) of Gemini 2.5 Pro",
      "inputTokenLimit": 1048576,
      "outputTokenLimit": 65536,
      "supportedGenerationMethods": [
        "generateContent",
        "countTokens",
        "createCachedContent",
        "batchGenerateContent"
      ],
      "temperature": 1,
      "topP": 0.95,
      "topK": 64,
      "maxTemperature": 2,
      "thinking": true
    }
  ],
  "nextPageToken": "Ch9tb2RlbHMvdmVvLTMuMS1nZW5lcmF0ZS1wcmV2aWV3"
}
```

4. 此外，当我们需要上传视频/多张图片等比较大的文件时，一般会经历“上传→处理→生成”三个步骤，我们需要加入像下面的代码中的“轮询等待”（Polling）逻辑。不然的话**代码会在上传完成后没有等待处理结束直接进入第三步**，此时文件还在PROCESSING状态而不是ACTIVE，所以API会出现类似`An error occurred during evaluation for videovista-engineering-3: 400 The File cirkjgqjssn2 is not in an ACTIVE state and usage is not allowed.`的错误

```python
            if processed_video_path != last_video_path or api_key != last_upload_api_key or video_file_object is None:
                try:
                    print(f"\nUploading video: {processed_video_path} with API Key {attempt_idx + 1}")
                    # genai.configure 已在 get_configured_client 中调用，设置了全局 active client

                    video_file_object = genai.upload_file(path=processed_video_path)

                    # 等待视频处理完成
                    print(f"Waiting for video processing: {video_file_object.name}")
                    while video_file_object.state.name == "PROCESSING":
                        time.sleep(5)
                        video_file_object = genai.get_file(video_file_object.name)
                        print(f"Current state: {video_file_object.state.name}")

                    if video_file_object.state.name == "FAILED":
                        print(f"Video processing failed: {video_file_object.name}")
                        continue # 当前Key上传失败，尝试下一个Key

                    # 上传成功，更新状态
                    last_video_path = processed_video_path
                    last_upload_api_key = api_key
                except Exception as e:
                    print(f"Error uploading video with Key {attempt_idx + 1}: {e}")
                    continue # 当前Key上传出错，尝试下一个Key
            else:
                print(f"\nUsing previously uploaded video for {question_id} (Key {attempt_idx + 1}).")
```

5. 以及，我们在处理过程中可能还会遇到类似`Invalid operation: The response.parts quick accessor requires a single candidate, but but response.candidates is empty.   This appears to be caused by a blocked prompt, see response.prompt_feedback: block_reason: PROHIBITED_CONTENT`的报错，这种问题一般来说不是代码层面的问题而是输入数据层面的问题，即**Google 的 AI 认为你提交的内容（主要是视频画面）违规了，拒绝生成回答，导致你的代码拿到了空结果，从而报错。** 我自己就遇到了一个这样的数据，试了很多次都是这样的，最后发现视频是一个女性在介绍她的各种泳衣以及实际穿着，被Google认为是色情内容ban掉了……虽然我咨询AI给出了让我显式关闭安全过滤的提示，但当我加入到代码后没有明显改善，那就这样吧……

6. 我在代码中加入了一个检查API KEY的逻辑。因为在我先前的代码中，虽然设置了API_KEY轮岗制度，但是却忽视了**API的权限隔离**问题。在上面我们提到了“上传→处理→生成”的三步走流程，在实际运行中，出现了“API 1成功上传了视频但是在推理过程中出现意外中断”的情况。在这种背景下，当我们直接替换为API 2处理问题时它没有执行上传那一步，而是直接从生成开始，所以遇到了`Error with API Key 2: 403 You do not have permission to access the File b1scxw08yc45 or it may not exist.`的问题，API Key无权访问Google服务器上的文件ID，因为它是由API Key 1上传的，文件私有，只有上传该文件的账号才有权限读取它。

7. 在代码的最后有一个time.sleep(10)逻辑，这是为了防止API访问速率过于频繁导致API被ban











