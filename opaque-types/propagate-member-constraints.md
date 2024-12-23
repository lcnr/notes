# Propagating opaque type constraints

Inside of the closure we end up with a defining use `Opaque<'arg0, 'arg1> = HiddenTy<'m0, 'm1>`. One idea would be to propagate this defining use to the parent.

To propagate this to the parent, it must only involve external regions. 

To propagate this constraint we need to map all regions to either `'static` or an external region. Unlike region constraints, member constraints have to mutate the region constraint graph.

We can eagerly require all member regions to be universal by requiring it to be alive for the whole body. This avoids having to think about relations between local regions.