Link to the github repository: https://github.com/eec289q-f23/project-2-screensaver-with-quadtree-Jack-already-taken

### Parallelization Attempt
I first used perf report on the original code to identify parts of the program most worthy of parallellization. Below is the perf report on which function calls contributes to most of the work
```
  42.00%  screensaver  screensaver        [.] intersectLines
  17.85%  screensaver  screensaver        [.] intersects
  15.20%  screensaver  screensaver        [.] intersect
  13.61%  screensaver  screensaver        [.] handle_intersections
   6.48%  screensaver  screensaver        [.] Vec_makeFromLine
   1.14%  screensaver  screensaver        [.] Vec_multiply
   0.92%  screensaver  screensaver        [.] Vec_add
   0.78%  screensaver  screensaver        [.] update_rectangles
   0.68%  screensaver  screensaver        [.] build_quad_tree
```
We can see that functions like ```intersectLines```, ```intersects```, ```intersect```, and ```handle_intersections``` are what contributes to most of the work, so this means parallelizing function calls in QuadTree.c that detects line intersection will give the best speedup.

To parallelize the intersection detection, we need to solve the issue of concurrent write to the ```IntersectionEventList```. I implement the cilk reducer which I used the given ```ien_merge_nodes``` function to merge two intersection even list. I then start parallelizing by spawning off recursive calls to ```handle_intersections``` function. Lower recursive depth of ```handle_intersections``` call tend to have higher work, since a quad tree based intersection detection need to test intersection for all line pairs within its current node and all its childern nodes. Therefore, I parallelized the double nested for loops for ```update_line_intersection_in_quad_tree``` and ```update_subline_intersections```. Instead of transforming those outer for loops into cilk_for, I implemented grain size for the for loops calculated at runtime using line like the following:
```c
int grain = BIN_LIMIT * BIN_LIMIT / collision_tree[index].num_lines;
```
This will ensure that for the parallelized for loops, each parallel process will have work load that's around the limit of ```BIN_LIMIT``` squared.

I also attpempted parallelizing quad tree construction, update line position, and calculate line wall collision, but they all resulted in worse runtime because the parallelizing gain is minimal compared to the parallel overhead.


### Optimization Attempt
The optimization attempt that I find most effective is tuning the ```BIN_LIMIT``` parameter for the quad tree, which specifies that the node of a quad tree should spawn of 4 childern nodes if it contains more number of lines than ```BIN_LIMIT```. I found that when pairing with my parallelization of intersection detection code, setting ```BIN_LIMIT``` to 28 gives the best runtime.

The rest of my optimization attempts weren't successful. I tried caching the vector length of the lines to speed up line length calculation in ```CollisionWorld_collisionSolver```. I tried to cache that data through adding another member in the Line struct and setting up an hash table of line lengths indexed by the line IDs, but both caching method resulted in a slowdown, which is likely due to increased cache misses. I also tried optimizing ```intersects``` function and ```intersect``` by merging the boolean expression and earlier exit for the case when there's no intersection, but these attempts resulted in either slowdown in runtime or incorrect results.

As far as I know, there's no error related to my current optimized code.

Compared to the original code's execution of 4000 frames of mit.in input, my optimized code has a speedup of around 7.33 times, and it is achieved by executing the code with 32 CPU cores, which is the maximum resource we could allocate for.

```console
$ srun -w agate-1 -t 1 --reservation=eec289q --cpu-bind=ldoms -c 32 ./screensaver 4000 input/mit.in
Number of frames = 4000
Input file path is: input/mit.in
---- RESULTS ----
Elapsed execution time: 0.833968s
1262 Line-Wall Collisions
19806 Line-Line Collisions
---- END RESULTS ----
```