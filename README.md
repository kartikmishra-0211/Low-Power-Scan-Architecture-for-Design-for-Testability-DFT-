# Low Power Scan Architecture for Design for Testability (DFT)

A Verilog study of how partitioning a scan chain reduces test-mode switching activity. A 4-bit accumulator is built twice: once with a single full scan chain (baseline), and once with the chain split into two sub-chains that a 2-bit counter enables one at a time.


## The idea

During scan shifting, every flip-flop in a long chain toggles on each clock, which raises dynamic power, IR drop and heat compared with normal operation. Splitting the chain and clocking only one segment at a time halves the flip-flops that can switch at any edge.

| | Phase 1: baseline | Phase 2: partitioned |
|---|---|---|
| Design under test | 4-bit accumulator, `Q <= Q + data_in` | Same |
| Scan chain | FF0 to FF1 to FF2 to FF3, one chain | Sub-chain 1 = FF0, FF1. Sub-chain 2 = FF2, FF3 |
| Control | `scan_en` only | 2-bit counter: counts 0 and 1 enable sub-chain 1, counts 2 and 3 enable sub-chain 2 |
| Flip-flops shifting per edge | 4 | 2 |
| Top module | `accumulator_scan` | `low_power_acc` |

The scan flip-flop is a Mux-D cell: with `se = 0` it captures `d` (functional mode), with `se = 1` it captures `si` (scan mode).

## Results (from the project report)

| Metric | Phase 1 | Phase 2 | Change |
|---|---|---|---|
| Dynamic power | 0.005 W | 0.004 W | 20% lower |
| Static power | 0.081 W | 0.081 W | none |
| Total power | 0.086 W | 0.085 W | about 1.2% lower |
| WNS at 100 MHz | | +8.0834 ns | timing met |
| Utilization | LUT 1%, FF 1%, IO 4% | LUT 1%, FF 1%, IO 4% | negligible |

Screenshots are in [docs/images](docs/images): schematics, simulation waveforms, power reports and utilization.

Note on precision: the power reports give values to 1 mW, so the 20% figure is a coarse estimate. In the report screenshots the saving shows up in the I/O portion of dynamic power (0.004 W to 0.003 W), and total power changes by only about 1%.

## Repository layout

```
src/
  scan_dff.v            scan_dff (Phase 1) and scan_dff_controlled (Phase 2)
  accumulator_scan.v    Phase 1: full scan accumulator
  low_power_acc.v       Phase 2: partitioned scan accumulator
constraints/
  clock.xdc             100 MHz clock constraint
docs/images/            schematics, waveforms, power and utilization screenshots
```

## How to run

1. Create a Vivado RTL project and add the three files in `src/` as design sources and `constraints/clock.xdc` as a constraint.
2. Choose the top module: `accumulator_scan` for Phase 1 or `low_power_acc` for Phase 2.
3. Write a small testbench that generates the clock, a reset pulse, `scan_en` and `scan_in` patterns and `data_in`. The testbench used for the report waveforms is not included in this repository.
4. Run behavioral simulation to check normal accumulation (`scan_en = 0`) and scan shifting (`scan_en = 1`), then run synthesis and implementation for timing, power and utilization.

## Notes on the code

- `scan_dff_controlled` is used by `low_power_acc`, but its source is not printed in the report appendix. It was written to match the behavior described in the report: it shifts only when its sub-chain is enabled, holds otherwise, and captures `d` normally when `scan_en = 0`.
- In `low_power_acc`, `reset` clears only the 2-bit counter. The `scan_dff_controlled` flip-flops start at 0 through an initial value. The Phase 1 flip-flops have an asynchronous reset.
- Both sub-chains take `scan_in` as their input, and `scan_out` is taken from FF1 or FF3 depending on the counter.

## Limitations

- Only a 4-bit example, so the power numbers are small and coarse.
- No ATPG or fault-coverage analysis was done. The work is a proof of concept for the architecture.
- Splitting chains adds routing and a small controller, which grows with the number of partitions.

## Future scope

Apply the same partitioning to a 32-bit or 64-bit design such as a RISC-V core, use an LFSR-based controller for built-in self-test (BIST), and choose the partitions automatically from the physical layout.

## License

MIT. See [LICENSE](LICENSE).
