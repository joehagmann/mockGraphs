<a href=https://hackmd.io/pYZUtHijSReXyjdEVDQyWQ><img src=https://hackmd.io/pYZUtHijSReXyjdEVDQyWQ/badge></a>

Please feel free to comment and collaborate using the link above. With signing in with your :octocat: github account, you can comment and edit.

<!--- create TOC with ~/sh/github-toc --->
* [Genome Graph Playground](#Genome-Graph-Playground)
  * [Background](#Background)
  * [Case 1: k11 and path-SGD](#Case-1-k11-and-path-SGD)
     * [Viz with Pantograph](#Viz-with-Pantograph)
  * [Case 2: k11 and Topo sort](#Case-2-k11-and-Topo-sort)
  * [Pantographs w/different parameters](#Pantographs-wdifferent-parameters)
* [Source Code](#Source-code)
      
      
# Genome Graph Playground

## Background

I seek out to understand the parameters for generating genome graphs to find their best  visualization by using a simple mockup MSA between three sequences.

Here are the sequences in an MSA above, and below the corresponding "most parsimonious" graph.

![](https://i.imgur.com/G0BA7kR.png)

Let's see how the latest graph generation software will build a graph from these sequences with using a few different parameter settings. We generate the graph using the tools `minimap2/seqwish(/smoothxg)/odgi`. Due to the small sequence lengths, `edyeet` cannot be used, so the single `pggb` steps are manually called and `minimap2` is used as mapping tool instead. We visualize the graphs with bandage and pantograph.

## Case 1: k11 and path-SGD
- minimap2: `-k11 -n1 -m8 -s10 -X -c`<br>
    - k-mer size: 11
    - minimal number of minimizers on a chain: 1
    - minimal chaining score: 8
    - minimal peak DP alignment score: 10
- seqwish: `-r10 -k10`
    - min-match-len: 10
    - repeat-max: 10
- odgi sort `-Y`
    - path guided linear 1D SGD (Gradient Descent)

With these parameters, the repeat sequences should align and stored only once in the graph, since they are longer than the k-mer size and min-match-length. Consequently, the seqwish graph in the bottom panel below looks basically like the expected one:

![](https://i.imgur.com/G0BA7kR.png)
![](https://i.imgur.com/RMJxzYp.png)

For smoothxg, we use a maximum block size of 20, i.e. larger than the red repeat. Default value in smoothxg is 10k, and this is at least for *A.thaliana* larger than the known repeat sequences of a few kb. So I guess this setting here should be quite realistic. (More params [below](#Pantographs-wdifferent-parameters))

The smoothed graph (`-w20 -c10`) gets rid of the single bp nodes (in the middle), and unrolls both the red and blue duplications:<br>
![](https://i.imgur.com/NAn08iW.png)
![](https://i.imgur.com/ilIL4Co.png)

smoothxg with `-w20 -c1000` (default minimum repeat length to collapse=1000, nothing should be collapsed) is similar to the one above (and equal to `-w10 -c10`): The repeat is again unrolled, and the substituted sequences are aligned on top of each other. The duplication occurs actually three times in the graph, since path 1 takes a completely other route than 2&3 because of the missing blue seqs:<br>
![](https://i.imgur.com/NAn08iW.png)
![](https://i.imgur.com/NwZaRNl.png)


**Question to Erik** smoothxg 'copies' instances of repeat sequences into the POA window where they are traversed, but loses the information across different POA windows, i.e. that the same sequence is traversed in multiple alignment windows. This is actually 'unrolling' as far as I understand. Is there a chance smoothxg can remember which parts of the seq also occurs somewhere else in the graph (and where)? This is basically a detection of repeated sequences, and in my opinion highly valuable information to visualize.


### Viz with Pantograph

The seqwish graph from above looks like this in Pantograph:

![](https://i.imgur.com/NAn08iW.png)
![](https://i.imgur.com/AEp2fYD.png)

**Issue 1** somewhere a circularization is introduced. The most left bins in the odgi output are the most right sequences in the MSA/fasta file. The first component thus fuses the last seqs and the first ones together.

It seems odgi viz shows the SNP after more than 14 bases from the start (left panel below), in the MSA the SNPs occurs at pos14. However, the inversion (red) is correctly placed at the end (right panel).<br>
<img src=https://i.imgur.com/PVujp3z.png width=250/>
<img src=https://i.imgur.com/ui1PMXA.png width=250/>


**Issue 2** The SGD sort tries to minimize node distances. Thus, because the duplicated sequence occurs at the beginning and the end of the sequences, it will be placed in the middle of the graph. For my taste, staying at one correct place is better, more intuitive and simplifies the structure.


## Case 2: k11 and Topo sort
- minimap2 and seqwish like in case 1
- odgi sort `-w` 
    - two-way (max of head-first and tail-first) topological sort

![](https://i.imgur.com/NAn08iW.png)
![](https://i.imgur.com/Syzdi27.png)

**Issue 3** Here, we see a problem with the re-use of nodes in the graph and thus 'components' in the pantograph visualization. It seems for sequence 1 that after the first instance of the duplication, we have to follow the green link to the most right conserved sequence block GGGCAT... This traversal is correct, but this link should be only followed in the second traversal through this component! With Pantograph you cannot yet distinguish routes when one pass follows a link and one other pass does not. A solution might be to trigger an unrolling in such cases in whichever viz-preparing step.

**Issue 4** The end of the sequences are somewhere in the middle of the matrix. We should mark that in the viz, as previously suggested by Simon.

Not sure I understand `odgi layout` correctly, but default options with `-C` yields a somewhat obscure graph:

<img src=https://i.imgur.com/PmKXUWy.png width=200/>


<!---
### Case 3: Smooth
- minimap2 and seqwish like in previous cases
- smoothxg: `-M -w20 -c10 -V`
    - merge contiguous MAF blocks in the MAF output and consensus sequences in the smoothed graph: true
    - maximum seed sequence in block: 20
    - minimum repeat length to collapse: 10

![](https://i.imgur.com/NAn08iW.png)
![](https://i.imgur.com/na5AhTw.png)

and the produced MSA below with the black background:

![](https://i.imgur.com/NAn08iW.png)

![](https://i.imgur.com/MDZJhhX.png)
--->

## Pantographs w/different parameters

- [minimap2] `-k` k-mer size
- [seqwish] `-k` min-match-len (Filter exact matches below this length. This can smooth the graph locally and prevent the formation of complex local graph topologies from forming due to differential alignments.)
- [seqwish] `-r` repeat-max (Limit transitive closure to include no more than N copies of a given input base)
- [smoothxg] `-w` maximum seed sequence in block
- [smoothxg] `-c` minimum repeat length to collapse


| case | minimap2 | seqwish     | sort | smoothxg      |
| ---- | -------- | ----------- | ---- | ------------- |
| def. | `-k11`   | `-k10 -r10` | Y    |               |
| 1    |          |             | w    |               |
| 2    |          |             |      | `-w20 -c10`   |
| 3    |          |             |      | `-w50 -c10`   |
| 4    |          |             |      | `-w10 -c10`   |
| 5    |          |             |      | `-w10 -c1`    |
| 6    |          |             |      | `-w10 -c1000` |
| 7    |          | `-k10 -r1`  | Y    |               |
| 8    |          | `-k30 -r10` | Y    |               |
| 9    |          | `-k30 -r10` | w    |               |
| 10   |          | `-k30 -r10` |      | `-w20 -c10`   |

![](https://i.imgur.com/NAn08iW.png)
seqwish `-k10` SGD: ![](https://i.imgur.com/AEp2fYD.png)
1 seqwish `-k10` topo: ![](https://i.imgur.com/Syzdi27.png)
2 smooth `-w20`, same as 3 `-w50`![](https://i.imgur.com/GudTV9s.png)
3 smooth `-w50`, same as 2 `-w20`![](https://i.imgur.com/DmbO1ka.png)
4 smooth`-c10`, same as 5 `-c1` and 6 `-c1000`![](https://i.imgur.com/hGG8hsk.png)
5 ![](https://i.imgur.com/exjVALY.png)
6 ![](https://i.imgur.com/DmGvHYD.png)
7 `-r1` senseless![](https://i.imgur.com/RUw2WEb.png)
8 seqwish `-k30` SGD ![](https://i.imgur.com/RkIhfIG.png)
9 seqwish `-k30` topo (path2 is wrong? bug?)![](https://i.imgur.com/MLh9tJh.png)
10 smooth `-k30 -w20` ![](https://i.imgur.com/akTHqSg.png)
![](https://i.imgur.com/NAn08iW.png)











## Other sorts

<p>
<details>

### k11 and depth-first sort
- minimap2 and seqwish like in cases 1 and 2
- odgi sort `-z`
    - depth-first sort
     
![](https://i.imgur.com/NAn08iW.png)
![](https://i.imgur.com/6JZOaFm.png)

Bad sorting. Even the SNP at the beginning is torn apart.

</p>
</details>



# Source code

<p>
<details>
Here are the sequences and the little bash script, for everyone to play around if you feel like it.

```
>1
AACGTACGATCGAGACTGCTAGACTTGATATGGACTGATGATAATACCGAGATACCTAGGAACAAAACGGTAGTGTGATATGGACTGATGATGGGCATATGCTAAACCTACGGGCAACC
>2
AACGTACGATCGATACTGCTAGACTTGATATGGACTGATGATGTCGTTATTGTTTTACAAAACGGTAGTGTTGGGTTAGGGTTGTGATAGAGGGATGATGGTATTGATATGGACTGATGATGGGCATATGCTGGTTGCCCGTAGGTTT
>3
AACGTACGATCGAGACTGCTAGACTTGATATGGACTGATGATAATACCGAGATACCTAGGAACAAAACGGTAGTGGTGGGTTAGGGTTGTGATATAGGGATGATGGTATTTGGGTTAGGGTTGTGATAGAGGGATGATGGTATTGATATGGACTGATGATGGGCATATGCTAAACCTACGGGCAACC
```

```
#!/bin/bash

mm_k=${1:-11}

sq_k=${2:-10}
sq_r=${3:-10}

sm_w=${4:-20}
sm_c=${5:-10}

mappings="mappings_MMk$mm_k"
graph="graph_MMk${mm_k}_SQk${sq_k}r${sq_r}"

# 1. minimap2
/ctx/projects/Q2380-Pantograph/software/minimap2/minimap2 -k$mm_k -X -o $mappings.paf -c -m8 -s10 -n1 assemblies.fa assemblies.fa
nr_mappings=`cat $mappings.paf | wc -l`
echo -e "\n---\nnr. mappings: $nr_mappings\n\n"
if [ $nr_mappings -lt 5 ]; then echo "will abort"; exit 0; fi

# 2. seqwish
seqwish -p $mappings.paf -s assemblies.fa -g $graph.gfa -r $sq_r -k $sq_k

# 3. odgi - prepare for visualization, compress and sort the graph
odgi build -g $graph.gfa -o $graph.og
odgi sort -i $graph.og -Y -o $graph.Ysorted.og
odgi viz -i $graph.Ysorted.og -o $graph.Ysorted.png -bg
odgi bin -i $graph.Ysorted.og -f $graph.Ysorted.pangenome.fa -j -w1 -sg > $graph.Ysorted.og.bin1.json

	odgi sort -i $graph.og -w -o $graph.wsorted.og
	odgi viz -i $graph.wsorted.og -o $graph.wsorted.png -bg
	odgi bin -i $graph.wsorted.og -f $graph.wsorted.pangenome.fa -j -w1 -sg > $graph.wsorted.og.bin1.json

#	odgi sort -i $graph.og -z -o $graph.zsorted.og
#	odgi viz -i $graph.zsorted.og -o $graph.zsorted.png -bg
#	odgi bin -i $graph.zsorted.og -f $graph.zsorted.pangenome.fa -j -w1 -sg > $graph.zsorted.og.bin1.json

# 4. smoothxg
graphSM="${graph}_SMw${sm_w}c${sm_c}"
smoothxg -g $graph.gfa -o $graphSM.gfa -m $graph.smooth.msa -M -w$sm_w -c$sm_c -V
odgi build -g $graphSM.gfa -o $graphSM.og
odgi viz -i $graphSM.og -o $graphSM.png -bg
odgi bin -i $graphSM.og -f $graphSM.pangenome.fa -j -w1 -sg > $graphSM.og.bin1.json

# 5. component segmentation
#conda activate pantograph
python /ctx/projects/Q2380-Pantograph/software/component_segmentation/segmentation.py -j $graph.Ysorted.og.bin1.json -f $graph.Ysorted.pangenome.fa -o ${graph}_Ysort
#viz_k11m8s10n1_k${sq_k}r10_Y
python /ctx/projects/Q2380-Pantograph/software/component_segmentation/segmentation.py -j $graph.wsorted.og.bin1.json -f $graph.wsorted.pangenome.fa -o ${graph}_wsort
#python /ctx/projects/Q2380-Pantograph/software/component_segmentation/segmentation.py -j $graph.zsorted.og.bin1.json -f $graph.zsorted.pangenome.fa -o ${graph}_zsort
python /ctx/projects/Q2380-Pantograph/software/component_segmentation/segmentation.py -j $graphSM.og.bin1.json -f $graphSM.pangenome.fa -o $graphSM

for i in ${graph}_Ysort ${graph}_wsort $graphSM; do
	echo "\"$i\",\"$i\"" > $i/dataset_entry
	scp -r $i ctxfs:/var/www/pantograph/test-data/
done

```

</details>
</p>