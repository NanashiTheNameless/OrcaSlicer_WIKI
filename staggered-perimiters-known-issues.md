# Staggered layers is currently in *Alpha*.

---

## If you find an issue or encounter a bug that is not on this list, please let us know by:
- [Opening an issue on this fork's GitHub repo](<https://github.com/NanashiTheNameless/OrcaSlicer>)

-----

# Known Issues:
- Staggared layers does not take in to account the slope of a wall, leading to `Inner-wall`s getting staggared even when visible from above.
- When `only_one_wall_first_layer` is **enabled**, the flowrate adjustment to correct for the lifted `Inner-wall`s is not correctly applied
- Having multiple models of different heights causes the check for the top layer to only work on the talest model (even if only 1 of the models has the setting enabled
- Orca slicer layer preview sees 1 layer as multiple different layers (usually 3, (lower walls, raised walls, infill). Sometimes 2, (lower walls and infill, raised walls)) (this is worsened if an object has 2 seperate sections of `outer-walls` as they get treated as seperate 'towers')
- Perimiters on internal holes are not staggered, only the outer most walls

# Incompatible settings: 
- `Adaptive Layer Height` is not currently supported
- `Initial_layer_print_height` & `Layer_height` must currently be the same
