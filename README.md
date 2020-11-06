Comment and collaborate here on ![HackMD](https://hackmd.io/pYZUtHijSReXyjdEVDQyWQ)

# Background

I seek out to understand the parameters for generating genome graphs to find their best  visualization by using a simple mockup MSA between three sequences.

Here are the sequences in an MSA:
![](https://i.imgur.com/D7Upylm.png)

This might be the corresponding "most parsimonious" graph and as such the most biologically meaningful graph(?) I assume, comments welcome if I am wrong.
![](https://i.imgur.com/EOD3XII.png)

Let's see how this graph is really represented using latest graph generation software: 
- `vg msga`
- `minimap2/seqwish/smoothxg`
- (due to the small sequence lengths, `edyeet` cannot be used, so the single `pggb` steps are manually called and `minimap2` is used as mapping tool instead)



