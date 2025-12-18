# Unreal-Engine-Vegetation-Growth-Shader
For my initial prototype, I experimented with a vegetation growth shader using a built-in UE5 asset. The goal was to rapidly prototype a growth effect using only native Unreal Engine tools. I explored two growth methods: one based on time, and another driven by camera distance. Both approaches focused on animating opacity from the root of each branch upward.
Due to the texture layout and directional variation in branch orientation, I introduced logic to handle growth in both forward and reverse directions.
Following this early test, I transitioned to a more structured pipeline to support broader asset compatibility. Using Houdini, I created a HIP file leveraging the SideFX Labs Tree Toolset to procedurally generate mesh variations via TOPs. I then embedded vertex color and UV data tailored to drive the UE5 shader, enabling consistent growth behaviour across multiple assets.

## How To Guide
1. Drag suitable asset from Content Browser into world<br>
  1.1. Under <b>Rendering</b> tab, set "Shadow Cache Invalidation Behaviour" to "Always" (this ensures the shadow updates)    
2. Add Material Instance (instance of MAT_VegetationGrowth) to asset

## Features
### Asset - Tree Asset Generation
The tree assets were created in Houdini using SideFX Labs Tree Toolset:
<img width="2210" height="862" alt="image" src="https://github.com/user-attachments/assets/c9cf5c6a-bbc7-4203-84ed-380152975a18" />

### Asset - Tree Asset Vertex Color 
The tree assets vertex color which drives the Vegetation Growth shader was then calculated based on branch position and length:
<img width="1081" height="927" alt="image" src="https://github.com/user-attachments/assets/fc68f627-16bb-4dd6-bbe4-5f2b04192bb7" />

### Shader - Shader Parameters
The parameters of the shader:
<img width="1160" height="703" alt="image" src="https://github.com/user-attachments/assets/ef76af05-1023-4bb1-992a-8a9766d986d2" />

## Deep Dive
### Asset - Tree Asset Generation
In order to quickly and efficiently generate multiple tree variants, a TOPs workflow was implemented where an <b>Attribute Name</b> was set and then used in the trunk/branches nodes to randomize each nodes output:
<img width="1457" height="661" alt="image" src="https://github.com/user-attachments/assets/47be46a7-8092-4f2c-b82c-0005aa625282" />
<img width="1268" height="732" alt="image" src="https://github.com/user-attachments/assets/61f44ce5-543d-4371-91fd-5ed332f76987" />

### Asset - Tree Asset Vertex Color
The vertex color is calculated based on the tree's branch positions and length. Looping through each "level" of branches, the vertex color of the previous iteration (branch level) feeds into the vertex color of the current branch level. This is first generated on the curves of the tree before being transferred to the mesh:
<img width="2049" height="1091" alt="image" src="https://github.com/user-attachments/assets/4fc35c36-5a07-4548-a0e6-e8beb2516652" />

Main VEX snippet (prims):
```c
//Initialse variables
vector prev_branch_vert_color;
int branch_level = detail(2, "iteration");
int max_branch_level = detail(2, "numiterations") - 1;
int prim_pts[] = primpoints(0, @primnum);
float branch_length = prim(0, "branch_length", @primnum);
float trunk_length = detail(0, "trunk_length");
float tree_height = detail(0, "tree_height");

float tree_end_factor = 0.5; //Unreal limit = 0.333, this allows 0.5>0.33 for leaves

//Determine trunk end factor
float factor = 1 - (trunk_length/tree_height);
float trunk_end_factor = fit(factor, 0, 1, 0.7, 0.95);

//Determine factor end for current branch
float f = float(branch_level)/float(max_branch_level);
float current_factor_end = lerp(trunk_end_factor, tree_end_factor, f);

//Sample prev branch vertex color
if(branch_level != 0)
{
    vector branch_start_pos = point(0, "P", prim_pts[0]);
    int prev_branch_ref_pt = nearpoint(1, branch_start_pos, 10);
    prev_branch_vert_color = point(1, "vertex_color", prev_branch_ref_pt);
}
else
{
    prev_branch_vert_color = {1,1,1};    
}

//Set vertex color
foreach(int pt; prim_pts)
{
    //Set variables
    vector vertex_color = point(0, "vertex_color", pt);
    float branch_length_factor = point(0, "branch_length_factor", pt);
    
    float alpha_factor = fit(branch_length_factor, 0, branch_length, prev_branch_vert_color.r, current_factor_end);
    vertex_color.r = alpha_factor;
    setpointattrib(0, "vertex_color", pt, vertex_color, "set");
}
```

## Setup
>[!NOTE]
> - Unreal Engine version - 5.7.0
> - Houdini version - 21.0.440

## Resources
- Tutorial for SideFX Labs Tree Toolset - https://www.sidefx.com/tutorials/tree-generator/
- Artstation Portfolio Piece - https://www.artstation.com/artwork/DLyLwE

## ToDo
- [x] Proof of Concept (Initial Prototype)
- [x] Feature - Create multiple types of trees, all with multiple variants
- [x] Feature - Use TOPs to efficiently generate the multiple variants of trees
- [ ] Feature - Expose and improve hidden "Time Based" growth option
- [ ] Debug Animation in Houdini with Curves
