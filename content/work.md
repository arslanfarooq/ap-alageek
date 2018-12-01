+++
title = "Work"
date = "2018-12-01"
menu = "main"
+++

Watch this space. I’ll get round to adding my work here soon enough.

In the meantime, here’s something from the guts of the rust compiler:

```
struct ReachableContext<'a, 'tcx: 'a> {
    // The type context.
    tcx: TyCtxt<'a, 'tcx, 'tcx>,
    tables: &'a ty::TypeckTables<'tcx>,
    // The set of items which must be exported in the linkage sense.
    reachable_symbols: NodeSet,
    // A worklist of item IDs. Each item ID in this worklist will be inlined
    // and will be scanned for further references.
    worklist: Vec<ast::NodeId>,
    // Whether any output of this compilation is a library
    any_library: bool,
}


impl<'a, 'tcx> Visitor<'tcx> for ReachableContext<'a, 'tcx> {
    fn nested_visit_map<'this>(&'this mut self) -> NestedVisitorMap<'this, 'tcx> {
        NestedVisitorMap::None
    }

    fn visit_nested_body(&mut self, body: hir::BodyId) {
        let old_tables = self.tables;
        self.tables = self.tcx.body_tables(body);
        let body = self.tcx.hir.body(body);
        self.visit_body(body);
        self.tables = old_tables;
    }

    fn visit_expr(&mut self, expr: &'tcx hir::Expr) {
        let def = match expr.node {
            hir::ExprPath(ref qpath) => {
                Some(self.tables.qpath_def(qpath, expr.id))
            }
            hir::ExprMethodCall(..) => {
                Some(self.tables.type_dependent_defs[&expr.id])
            }
            _ => None
        };

        if let Some(def) = def {
            let def_id = def.def_id();
            if let Some(node_id) = self.tcx.hir.as_local_node_id(def_id) {
                if self.def_id_represents_local_inlined_item(def_id) {
                    self.worklist.push(node_id);
                } else {
                    match def {
                        // If this path leads to a constant, then we need to
                        // recurse into the constant to continue finding
                        // items that are reachable.
                        Def::Const(..) | Def::AssociatedConst(..) => {
                            self.worklist.push(node_id);
                        }

                        // If this wasn't a static, then the destination is
                        // surely reachable.
                        _ => {
                            self.reachable_symbols.insert(node_id);
                        }
                    }
                }
            }
        }

        intravisit::walk_expr(self, expr)
    }
}
```

