# Transformer Implementation


```python
import torch
import torch.nn as nn
import numpy as np
```

## How to calculate self-attention

$Y = softmax(\frac{KQ^T}{\sqrt(d})V $

$K = XW_k$, $Q = XW_Q$, $V = XW_V$

## Simple implementation without batch_size


```python
def softmax(Z):
  Z = np.exp(Z - Z.max(axis=-1,keepdims=True))
  Z = Z / Z.sum(axis=-1,keepdims=True)
  return Z

def self_attention(X,mask,W_KQV,W_out): #no batch_size version
  K,Q,V = np.split(X@W_KQV,3,axis=-1)
  attn = softmax(K@Q.T / np.sqrt(X.shape[1])+mask)
  return attn@V@W_out, attn

```


```python
T,d = 100,64
attn = nn.MultiheadAttention(d,1,bias=False,batch_first = True) # 1 represents the number of heads
M = torch.triu(-float("inf") * torch.ones(T,T),1)
X = torch.randn(1,T,d)
Y_,A_ = attn(X,X,X,attn_mask = M)
```


```python
Y_.shape
```


    torch.Size([1, 100, 64])


```python
attn.in_proj_weight.shape
```


    torch.Size([192, 64])


```python
attn.out_proj.weight.shape
```


    torch.Size([64, 64])


```python
Y,A = self_attention(X[0].numpy(), M.numpy(),attn.in_proj_weight.detach().numpy().T, attn.out_proj.weight.detach().numpy().T)
```


```python
np.linalg.norm(Y_[0].detach().numpy() - Y)
```


    1.9094748e-06

## Implementation with batch_size


```python
def self_attention(X,mask,W_KQV,W_out): #no batch_size version
  K,Q,V = np.split(X@W_KQV,3,axis=-1)
  attn = softmax(K@Q.swapaxes(-1,-2) / np.sqrt(X.shape[-1])+mask)
  return attn@V@W_out, attn
```


```python
B,T,d = 50,100,64
X = torch.randn(B,T,d)
M = torch.triu(-float("inf") * torch.ones(T,T),1)
Y_,A_ = attn(X,X,X,attn_mask = M)
```


```python
attn.in_proj_weight.shape
```


    torch.Size([192, 64])


```python
Y,A = self_attention(X.numpy(), np.broadcast_to(M.numpy(),(B,T,T)),attn.in_proj_weight.detach().numpy().T, attn.out_proj.weight.detach().numpy().T)
```


```python
np.linalg.norm(Y_.detach().numpy() - Y)
```


    1.2184518e-05

## Multihead


```python
def multihead_attention(X,mask,heads,W_KQV,W_out):
  B,T,d = X.shape
  K,Q,V = np.split(X@W_KQV,3,axis = -1)
  K,Q,V = [a.reshape(B,T,heads,d//heads).swapaxes(1,2) for a in (K,Q,V)]
  attn = softmax(K@Q.swapaxes(-1,-2)/ np.sqrt(d//heads)+mask)
  return (attn@V).swapaxes(1,2).reshape(B,T,d)@W_out,attn

```


```python
heads = 4
attn = nn.MultiheadAttention(d,heads,bias=False,batch_first=True)
Y_,A_ = attn(X,X,X,attn_mask = M)
```


```python
A_.shape
```


    torch.Size([50, 100, 100])


```python
Y,A = multihead_attention(X.numpy(), M.numpy(),heads,attn.in_proj_weight.detach().numpy().T, attn.out_proj.weight.detach().numpy().T)
```


```python
A.shape
```


    (50, 4, 100, 100)


```python
np.linalg.norm(Y_.detach().numpy() - Y)
```


    1.1593343e-05

