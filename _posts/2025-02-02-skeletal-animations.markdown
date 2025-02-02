---
layout: post
title:  "So you want to implement Skeletal Animation"
date:   2025-02-02 16:00:00 +0100
categories: rendering engine
hidden: false
---

This is the first entry in a series of posts i'm doing about implementing Skeletal Animation.
The introduction will be updated with links to the next posts

### SMALL BUT NEEDED INTRODUCTION ON SKELETAL ANIMATION
Video games are composed of multiple layers to immerse the player into their worlds: one of these layers (and imho one of the most important ones) is character animation.

One of the ways to bring our characters to life is through *Skeletal animation*: we attach a mesh's vertices to a *skeleton*, which approximates the mesh: such meshes are called **Skinned meshes** (the opposite of a Skinned Mesh is often called Static Mesh). 

The skeleton is usually handmade to suit the mesh through a process called *rigging*: after the character is modeled, the artist places some *bones* (or *joints*) on the mesh in a hierarchy, where the topmost bone (that is the one parent to them all) is called the skeleton's **root bone**: the bones placed in this phase define the model's *rest pose*, that is where the vertices lie when no animation is playing.

> The reason the rest pose is important is that an artist might model a character in an A pose (with the arms pointing down), but, for ease of animation, the rest pose might be a T pose.

INSERT SKELETON PIC

Each vertex in the mesh is then bound to one or more bones through a process called *weight painting*, where the artist "paints" on the mesh how much each bone influences a certain vertex (this process can also be automated, e.g through Blender's _Automatic Weights_ feature).

INSERT BLENDER WEIGHT PAINTING SCREEN

After all this is done, animations can be created: each animation is a group of **key poses**, which define the most important poses for the animation.
The key poses are then interpolated through one of more curves over a certain time span: these poses define for some of the skeleton's bones their transform in 3D.
Sampling these curves results in a *Keyframe*, which is then applied to the mesh during rendering.

INSERT BLENDER ACTION EDITOR PIC

### GETTING OUR HANDS DIRTY
Let's start by defining a Static Mesh's `Vertex`
```c
struct Vertex {
    vec3 position;
    vec3 normal;
    vec2 tex_coords;
	
    vec3 tangent;
    vec3 bitangent;
};
```
A Skinned Mesh is composed of Skinned Vertices (duh): a Skinned Vertex just extends our `Vertex` struct with two additional fields: one indicating which bones the vertex is influenced from, and another indicating how much the bone influences the vertex.
```glsl
struct SkinnedVertex {
    vec3 position;
    vec3 normal;
    vec2 tex_coords;
	
    vec3 tangent;
    vec3 bitangent;

    ivec4 bone_ids;
    vec4 bone_weights;
};
```
where the sum of `bone_weights` should equal to 1.

You can see that, with this definition, we're limited to up to 4 bones per vertex: this is usually enough, and should suffice for most applications.

As we said earlier, we will be appling a specific **Keyframe** during rendering: what we mean is that the keyframe will contain for each bone a model space affine transform matrix for each bone, which we will use to transform each vertex.
For now, let's assume that we got from some magic place an array of `mat4 bone_transforms` containing the keyframe's bone transform matrices, and let's define the set of operations needed to animate a Skinned Mesh.

The algorithm is very simple:
1. We fetch one of the Skinned Mesh's vertices
2. We iterate on the vertex's influencing bone indices
3. For each bone, we transform the vertex's position with the bone transform, accumulating the positions.
4. The accumulated position is the result of the skinning process

My engine does something like this
```glsl
mat4 bone_transforms[MAX_BONES]; // the array of bone transforms
struct TransformedVertex {
	vec3 position;
	vec3 normal;
};

TransformedVertex compute_skinned_position(SkinnedVertex v) {
	TransformedVertex s;
	for(int i = 0; i < max_bones; ++i) {
		if(is_invalid_bone_id(v.bone_ids[i])) {
			break;
		}
	
		mat4 bone_transform = bone_transforms[v.bone_ids[i]];
		s.position += v.bone_weights[i] * (bone_transform * vec4(v.position, 1.0f));
		s.normal += v.bone_weights[i] * (bone_transform * vec4(v.normal, 0.0f));
	}

  s.normal = normalize(s.normal); // Always remember to normalize any surface vector

	return s;
}
```
Now, you're probably wondering: how do we fill `bone_transforms`?

### DEALING WITH TRANSFORMATIONS
When we rotate our arm, it makes sense to define how our arm rotates with respect to our shoulder: in the same way, we define our hand's rotation w.r.t our forearm, which is then rotated w.r.t our arm, which is then rotated w.r.t our shoulder and so on, until we reach our spine (or, in the case of our mesh's skeleton, the root bone).

Animatons work in the same way: for each keyframe, we know the bone's *local transform* (expressed w.r.t the bone's parent): in order to know each bone's global transform, we need to multiply the bone's local transform with the bone parent's global transform.

Expressed in pseudocode, it's something like this
```c
mat4 global_transform(Bone bone) {
  if(bone has no parent) {
    return bone.local_transform;
  }
  return bone.local_transform * global_transform(bone.parent);
}
```

> A keen eye might noticed that with the pseudocode above we're computing more matrix transformations than needed: we will deal with this issue in the next post.

#### THE INVERSE BIND POSE MATRIX
Exceeeept... we're not done yet.

Remember how we defined the mesh's *rest pose* earlier? Yeah, that's a transformation we need to account for.
When the rest pose is applied, all the vertices are transformed by the influencing bones' *bind pose matrices*, to transform the bones from model space in bone local space.
That's an issue, because in our applications we're most likely working in model space: thus, we need to undo the bind pose transformation.
With all said, that's very easy: the bind pose matrix is just another affine transformation matrix. We can then apply the inverted matrix, often called a bone's *inverse pose matrix*, to the result of our `global_transform()` function:
Given a `Bone b`
```c
bone_transforms[b.id] = bone.inverse_bind_pose * global_transform(b);
```

The last remaining piece of the puzzle is reading the local transforms for each animation.

