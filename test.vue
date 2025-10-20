FROM ./你的模型文件夹路径

SYSTEM """You are DeepSeek Coder, a helpful AI programming assistant."""

TEMPLATE """{{ if .System }}<|system|>
{{ .System }}<|end|>
{{ end }}<|user|>
{{ .Prompt }}<|end|>
<|assistant|>
"""

PARAMETER stop "<|end|>"
PARAMETER stop "<|user|>"