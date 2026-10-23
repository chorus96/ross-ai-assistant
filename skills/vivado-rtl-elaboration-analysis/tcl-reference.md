<!-- ko-translation: enabled -->
# TCL Reference — RTL Elaboration Analysis
# TCL 참조 - RTL 정교화 분석

Reference TCL blocks for MCP mode workflow.
MCP 모드 워크플로에 대한 참조 TCL 블록입니다.
Execute all commands through the
모든 명령을 다음을 통해 실행합니다.
`vivadoExecute` tool — **never** create `.tcl` script files.
`vivadoExecute` 도구 — **절대로** `.tcl` 스크립트 파일을 생성하지 마세요.

---

## Step 1: Detect Project Mode
## 1단계: 프로젝트 모드 감지

```tcl
set project_path [pwd]

set xpr_files [glob -nocomplain *.xpr]

if {[llength $xpr_files] > 0} {
    # PROJECT MODE
    set project_file [lindex $xpr_files 0]
    open_project $project_file
    set part_number [get_property part [current_project]]
    set project_dir [get_property DIRECTORY [current_project]]
    set top_module [get_property top [current_fileset]]
    if {$top_module eq ""} {
        set all_tops [find_top]
        set top_module [lindex $all_tops 0]
        if {[llength $all_tops] > 1} {
            puts "WARNING: Multiple top-module candidates: $all_tops"
            puts "  Auto-selected: $top_module"
        } elseif {$top_module eq ""} {
            error "Could not auto-detect top module."
        }
    }
    puts "PROJECT mode: $project_file | Top: $top_module | Part: $part_number"
} else {
    # NON-PROJECT MODE
    set rtl_files [glob -nocomplain *.v *.sv *.vhd *.vhdl]
    if {[llength $rtl_files] == 0} {
        error "No RTL files found in workspace"
    }
    set part_number "xc7k70tfbg676-2"
    set project_dir [pwd]

    # Read files with correct language flags
    foreach f [glob -nocomplain *.sv] { read_verilog -sv $f }
    foreach f [glob -nocomplain *.v]  { read_verilog $f }
    foreach f [glob -nocomplain *.vhd *.vhdl] { read_vhdl $f }

    set all_tops [find_top]
    set top_module [lindex $all_tops 0]
    puts "NON-PROJECT mode | Top: $top_module | Part: $part_number"
}
```

---

## Step 2: Run RTL Elaboration
## 2단계: RTL 정교화 실행

```tcl
# Create report directory
file mkdir vivado_agentic_ai_reports/rtl-elaboration-analysis

# Run elaboration only (-rtl) — much faster than full synthesis
# This triggers all Verific front-end messages without running
# synthesis optimization/mapping phases
catch {synth_design -top $top_module -part $part_number -rtl -name rtl_1} elab_result
puts "synth_design -rtl result: $elab_result"

# Optional speed-up switches:
#   -rtl_skip_ip          Skip loading OOC IP DCPs (uses stubs)
#   -rtl_skip_constraints  Skip loading XDC constraints
#   -rtl_skip_mlo          Skip MLO processing

# The elaboration messages are in the Vivado log
# For non-project mode: vivado.log in cwd
# For project mode: vivado.log (since -rtl doesn't use launch_runs)
```

---

## Step 3: Locate Log File
## 3단계: 로그 파일 찾기

```tcl
# synth_design -rtl writes to vivado.log in cwd (both project and non-project mode)
set log_file "vivado.log"

if {![file exists $log_file]} {
    error "Log file not found: $log_file"
}
puts "Log file: $log_file"
```

---

## Notes
## 메모

- `synth_design -rtl` runs only the Verific front-end (parsing + elaboration),
- `synth_design -rtl`은 Verific 프런트엔드(파싱 + 정교화)만 실행합니다.
skipping synthesis optimization and technology mapping — typically 5-10x faster
합성 최적화 및 기술 매핑 건너뛰기 - 일반적으로 5~10배 더 빠름
- Even if elaboration fails (returns error), the log will contain all
- 정교화가 실패하더라도(오류 반환) 로그에는 모든 내용이 포함됩니다.
messages up to the failure point
실패 지점까지의 메시지
- The `catch` wrapper ensures we can still access the log after errors
- `catch` 래퍼는 오류 후에도 로그에 계속 액세스할 수 있도록 보장합니다.
- Use `-rtl_skip_ip` for large designs with many OOC IP to speed up further
- OOC IP가 많은 대규모 설계에는 `-rtl_skip_ip`를 사용하여 속도를 더욱 높이세요.
- If full synthesis log already exists (e.g., user provides a `runme.log`),
- 전체 합성 로그가 이미 존재하는 경우(예: 사용자가 'runme.log' 제공)
the log-file-only workflow is preferred — no need to re-run elaboration
로그 파일 전용 워크플로가 선호됩니다. 정교화를 다시 실행할 필요가 없습니다.
