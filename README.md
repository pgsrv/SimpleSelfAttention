# SimpleSelfAttention (5/14/2019)
Python 3.7, Pytorch 1.0.0, fastai 1.0.52


The purpose of this repository is two-fold:
- demonstrate improvements brought by the use of a self-attention layer in an image classification layer
- introduce a new layer which I call SimpleSelfAttention

## Image classification results

We evaluate our model on the Imagenette/Imagewoof datasets [1]. We compare it to a baseline xresnet50 model[2], which is currently the best model on the Imagenette/Imagewoof leaderboards (as of 5/14/2019).

### Preliminary results (5/14/2019):

| Dataset | Image Size  |  Epochs | Baseline avg accuracy  | Proposed model avg accuracy | GPUs  |
|---|---|---|---|---|---|
| Imagewoof | 256  |  5 |  61.9% (10 runs)| 67.6% (10 runs)  | 1  |
| Imagewoof | 256 | 20  |  84.6% (1 run) |  85.7% (2 runs) | 1 |
|  Imagewoof | 256  |  80 | 89%???  | 89.6% (1 run)  | 1 |
|  Imagewoof | 256  |  400 |  90.2%??? | 91% (1 run)  | 1 |

There needs to be more runs on both baseline and proposed models for 20 and more epochs. Also, I have some doubts on the baseline accuracy originally reported ([1]), as our baseline results on 5 and 10 epochs are higher than the original ones.


## Simple Self Attention layer

The only difference between baseline and proposed model is the addition of a self-attention layer at a specific position in the architecture. Other positions have been tested with worse results. Also, adding multiple self-attention layers has made results worse. 

The new layer, which I call SimpleSelfAttention, is based on the fastai implementation ([3]) of the self attention layer described in the SAGAN paper ([4]).

#### Original layer:

          
     class SelfAttention(nn.Module):
    
      "Self attention layer for nd."

      def __init__(self, n_channels:int):
          super().__init__()
          self.query = conv1d(n_channels, n_channels//8)
          self.key   = conv1d(n_channels, n_channels//8)
          self.value = conv1d(n_channels, n_channels)
          self.gamma = nn.Parameter(tensor([0.]))

      def forward(self, x):
          #Notation from https://arxiv.org/pdf/1805.08318.pdf
          size = x.size()
          x = x.view(*size[:2],-1)
          f,g,h = self.query(x),self.key(x),self.value(x)
          beta = F.softmax(torch.bmm(f.permute(0,2,1).contiguous(), g), dim=1)
          o = self.gamma * torch.bmm(h, beta) + x
          return o.view(*size).contiguous()

#### Proposed layer:
      
    class SimpleSelfAttention(nn.Module):
    
    def __init__(self, n_in:int, ks=1):#, n_out:int):
        super().__init__()            
        self.conv = conv1d(n_in, n_in, ks, padding=ks//2, bias=False)             
        self.gamma = nn.Parameter(tensor([0.]))      
     
    def forward(self,x):               
        size = x.size()
        x = x.view(*size[:2],-1)
        o = torch.bmm(x.permute(0,2,1).contiguous(),self.conv(x))      
        o = self.gamma * torch.bmm(x,o) + x      
         
        return o.view(*size).contiguous()   


As described in the SAGAN paper ([4]), the original layer takes the image features x of shape (C,N) (where N = H * W), and transforms them into f(x) = Wf * x and g(x) = Wg * x, where Wf and Wg have shape (C,C'), and C' is chosen to be C/8. Those matrix multiplications can be expressed as (1 * 1) convolution layers. Then, we compute S = (f(x))^T * g(x).
Therefore, S = (Wf * x)^T * (Wg * x) = x^T * (Wf ^T * Wg) * x. My first proposed simplification is to combine (Wf ^T * Wg) into a single (C * C) matrix W. So S = x^T * W * x.  S = S(x,x) (bilinear form) is of shape (N * N) and will represent the influence of each pixel on other pixels ("the extent to which the model attends to the ith location when synthesizing the jth region" [4]). Note that S depends on the input, but that W 

Therefore, we only learn weights W for one convolution layer instead of weights Wf and Wg for two convolution layers. Advantages are: simplicity, removal of one hyperparameter (C' = C/8), and a matrix W that offers more possibilities than Wf ^T * Wg. One possible drawback is that we have more parameters to learn (C^2 vs C^2/4). One option we haven't tried here is to force W to be a symmetrical matrix. This would reduce the number of parameters and force the influence of "pixel" j on pixel i to be the same as pixel i on pixel j.

The next step in the original version of the layer is to compute the softmax of matrix S. I decided to remove this step completely and work with unrestricted weights instead of normalized probability-like weights.

The final step in the original version is to compute h(x) = Wh * x (Wh of shape (C * C)), which is also implemented as a 1 * 1 convolution layer. Then our final output is o = gamma * h(x) * S + x.  We propose to remove this final convolution layer and have the output be o = gamma * x * S + x. This final convolution can be re-added as a separate layer, although this implies a different position for the skip connection.





# References

[1] https://github.com/fastai/imagenette
[2] https://github.com/fastai/fastai/blob/master/examples/train_imagenette.py
[3] https://github.com/fastai/fastai/blob/5c51f9eabf76853a89a9bc5741804d2ed4407e49/fastai/layers.py
[4] https://arxiv.org/abs/1805.08318