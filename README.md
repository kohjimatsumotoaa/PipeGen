%md
### 2CrvSTL2Size
#### 高レベル処理フロー（STL→中心線→2弧検出→接線整合→θ確定→出力）
flowchart TD  

    A[入力: STLファイルパス] --> B[STL読み込み: numpy-stl]  
    B --> C[頂点・法線ユニーク化<br/>unique_vertices_with_normals]  
    C --> D[管半径の決定<br/>estimate_tube_radius_local_pca<br/>(known_tube_radius優先)]  
    D --> E[中心線候補の生成<br/>offset_centerline_points<br/>(+/-法線方向へ半径分オフセット)]  
    E --> F[ダウンサンプル & kNNグラフ]  
    F --> G[最長路抽出<br/>longest_path_on_knn]  
    G --> H[等間隔再サンプル<br/>resample_polyline_by_arclength]  
    H --> I[検出用平滑化(任意)<br/>Savitzky-Golay]  
    I --> J[曲率算出 & 平滑化<br/>discrete_curvature + moving average]  
    J --> K{2つの曲線区間を検出<br/>find_two_curved_segments_adaptive}  
    K -- 失敗 --> K2[窓拡大・閾緩和で再試行]  
    K2 -- 失敗 --> ERR[[エラー: 曲線セグメントが2つ取れない]]  
    K -- 成功 --> L1[弧1の残差ゲート・円フィット<br/>refine_arc_by_residual]  
    K -- 成功 --> L2[弧2の残差ゲート・円フィット<br/>refine_arc_by_residual]  
    L1 --> M1[位相単調性で弧を拡張<br/>expand_arc_by_monotonic_phi]  
    L2 --> M2[位相単調性で弧を拡張<br/>expand_arc_by_monotonic_phi]  
    M1 --> N1[端点連続最適化(座標降下＋交点候補)<br/>optimize_arc_endpoints_continuous]  
    M2 --> N2[端点連続最適化(座標降下＋交点候補)<br/>optimize_arc_endpoints_continuous]  
    N1 --> T[接線整合: 直線↔円の接線一致で端点再揃え<br/>enforce_tangent_continuity]  
    N2 --> T  
    T --> U1[θ確定(ハイブリッド)<br/>finalize_theta_hybrid (弧1)]  
    T --> U2[θ確定(ハイブリッド)<br/>finalize_theta_hybrid (弧2)]  
    U1 --> V[セグメント構築<br/>S1 / C1 / S2 / C2 / S3]  
    U2 --> V  
    V --> W[結果出力<br/>半径・中心線・各S/C長/弧半径/角度]  
    W --> Z[ログ表示・CLI出力]  

#### 詳細フロー（主な関数と入出力の関係）
flowchart LR  

    subgraph Preprocessing[前処理]  
        P1[mesh.Mesh.from_file] --> P2[unique_vertices_with_normals\n入力: STLメッシュ\n出力: unique頂点, 法線]  
        P2 --> P3{半径決定}  
        P3 -- knownあり --> P3a[known_tube_radiusを使用]  
        P3 -- knownなし --> P3b[estimate_tube_radius_local_pca]  
        P3a --> P4  
        P3b --> P4[offset_centerline_points\n入力: verts, normals, 半径\n出力: 中心線候補点群C]  
        P4 --> P5[kNNグラフ → 最長路\nlongest_path_on_knn\n出力: P0(ポリライン順序点群)]  
        P5 --> P6[resample_polyline_by_arclength\n出力: P(等間隔中心線)]  
        P6 --> P7[Savitzky-Golay平滑(任意)\n出力: P_det]  
        P7 --> P8[discrete_curvature + 平滑\n出力: kappa_s]  
    end  

    subgraph Detection[曲線区間検出]  
        P8 --> D1[find_two_curved_segments_adaptive\n入力: P_det, kappa_s\n出力:   (c1_i0_raw, c1_i1_raw), (c2_i0_raw, c2_i1_raw)]  
        D1 -- 失敗 --> D1b[窓拡大・閾緩和再試行]  
        D1b -- 失敗 --> ERR[[停止]]  
    end  

    subgraph ArcRefine[弧の円フィット・残差ゲート]  
        D1 --> R1[refine_arc_by_residual (Arc1)\n出力: c1_i0, c1_i1, r1, plane1]
        D1 --> R2[refine_arc_by_residual (Arc2)\n出力: c2_i0, c2_i1, r2, plane2]
        R1 --> E1[expand_arc_by_monotonic_phi (Arc1)\n出力: c1_i0_phi, c1_i1_phi]
        R2 --> E2[expand_arc_by_monotonic_phi (Arc2)\n出力: c2_i0_phi, c2_i1_phi]
        E1 --> O1[optimize_arc_endpoints_continuous (Arc1)\n出力: c1_i0_opt, c1_i1_opt, s1_0, s1_1]
        E2 --> O2[optimize_arc_endpoints_continuous (Arc2)\n出力: c2_i0_opt, c2_i1_opt, s2_0, s2_1]
    end

    subgraph TangentContinuity[接線整合]  
        O1 --> T1[enforce_tangent_continuity\n入力: P, s, plane1/r1, plane2/r2,\n(c1_i0_opt, c1_i1_opt, c2_i0_opt, c2_i1_opt)\n出力: c1_i0_tc, c1_i1_tc, c2_i0_tc, c2_i1_tc]  
        O2 --> T1  
        T1 --> S1[s更新: s[c1_i0_tc], s[c1_i1_tc], s[c2_i0_tc], s[c2_i1_tc]]  
    end  

    subgraph ThetaFinalize[θ確定(ハイブリッド)]  
        S1 --> F1[finalize_theta_hybrid (Arc1)\n入力: P, plane1, r1, c1_i0_tc, c1_i1_tc\n出力: θ1, meta1]  
        S1 --> F2[finalize_theta_hybrid (Arc2)\n入力: P, plane2, r2, c2_i0_tc, c2_i1_tc\n出力: θ2, meta2]  
    end  

    subgraph Output[出力整形]  
        F1 --> OUT1[Segments構築: S1/C1/S2/C2/S3\n長さ・半径・角度]  
        F2 --> OUT1  
        OUT1 --> OUT2[Result dict\n(tube_radius, centerline,\nsegments, straight_lengths,\ncurve_radii, curve_angles_rad/deg)]  
        OUT2 --> CLI[ログ/CLI表⽰]  
    end  

#### 接線整合の内部（S↔C境界端点の再位置合わせ）
flowchart TD

    A1[fit_line_on_straight_window (S1左窓/S2右窓/S3右窓)] --> A2[fit_line_ransacでローカル直線推定]  
    A2 --> B1[line_to_plane_2dで2D化]  
    B1 --> B2[tangent_points_on_circle_for_lineで円の接線接点を理論計算]  
    B2 --> C1[接点を3Dに戻す→ポリラインへ最近接投影\nproject_point_onto_polyline_arclength]  
    C1 --> D1[最も境界近い投影点を採用\n→ 弧端点インデックス再設定]  
    D1 --> E1[境界順序整合性のチェック/補正]  

#### ハイブリッドθ確定の内部（複数候補を整合性で選択）
flowchart LR  

    X0[弧 i0..i1 と s0..s1] --> X1[robust_phase_regression_theta\n(残差ゲート + 位相回帰)]  
    X0 --> X2[phi_at_s_bandでφ帯域平均]  
    X0 --> X3[tangent_angle_on_arcで接線角差]  
    X1 --> Y1[θ_slope]  
    X2 --> Y2[θ_band]  
    X3 --> Y3[θ_tan]  
    Y1 --> Z1{整合性: |r*θ - L|/L}  
    Y2 --> Z1  
    Y3 --> Z1  
    Z1 -- 最良候補(≦3%) --> θ1[θ採用 + meta]  
    Z1 -- 不十分 --> R1[residual_gate_indices + refit_circle_on_indices\n(plane2, r2)]  
    R1 --> 再計算[同様の候補(Y1/Y2/Y3)を再評価]  
    再計算 -- 良好 --> θ2[θ採用 + meta]  
    再計算 -- 依然不十分 --> θ3[フォールバック: L/r2 または L/r]  

#### 補足

- 検出の堅牢性  
  ○ 曲率ベースの2弧検出で失敗した場合、平滑窓拡大・最小曲線長緩和で再試行します。
- 弧端点の安定化  
  ○ 残差ゲート（円フィットの半径残差）＋位相単調性（φの単調増減）で弧の実体を拡張し、短縮やズレを抑制します。  
  ○ 連続端点最適化は、L/r・残差・位相勾配の複合コストを座標降下＋交点候補（直線×円の交点）で最小化。
- 接線整合（tangent continuity）  
  ○ 直線側のローカルRANSAC直線の方向と円の接線方向が一致する円周の接点を理論的に求め、ポリラインへ投影して弧端点を再位置合わせ。  
  ○ これにより S↔C 境界で接線連続となり、直線長とθの系統誤差をさらに低減します。
- θのハイブリッド確定  
  ○ 位相回帰(θ_slope)、帯域平均(θ_band)、接線角差(θ_tan)の3候補を「長さ整合性」で比較し、最良を採用。  
  ○ 必要に応じて残差ゲート再フィットで平面・半径を更新して再評価、最終的にL/rフォールバックも用意。