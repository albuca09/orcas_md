# Simulação completa de hélice naval no Unreal Engine

## Objetivo
Criar uma simulação em tempo real de uma hélice de navio girando em um meio líquido, com controle de parâmetros como:

- Pressão
- Densidade
- Temperatura
- Salinidade
- Velocidade de rotação (RPM)
- Geometria da hélice (diâmetro, passo, número de pás)

E analisar efeitos físicos de fluidodinâmica/hidrodinâmica, como:

- Empuxo (thrust)
- Torque no eixo
- Eficiência propulsiva
- Número de Reynolds
- Número de cavitação (σ)
- Início de cavitação (estimado)

---

## 1) Decisão de arquitetura (importante)

No Unreal Engine, você tem **duas abordagens** principais:

1. **Simulação em tempo real aproximada (recomendada para interatividade):**
   - Modelo físico simplificado/empírico.
   - Roda em FPS alto.
   - Ideal para testar cenários, interfaces, controle e visualização.

2. **CFD de alta fidelidade (off-line):**
   - Resolve Navier-Stokes com malha refinada.
   - Usa OpenFOAM/Ansys/STAR-CCM+ etc.
   - Muito mais caro computacionalmente.

### Recomendação prática
Use o Unreal como **digital twin interativo** e faça acoplamento com:

- **Modelo reduzido** interno para tempo real.
- **Resultados CFD pré-computados** (lookup tables/surrogates) para ganhar realismo.

---

## 2) Componentes do sistema

### 2.1 Cena e atores

- `BP_PropellerSystem` (ator principal)
  - `StaticMeshComponent` da hélice
  - `SceneComponent` do eixo
  - `UPropellerHydroComponent` (componente custom C++ para física)
  - `UDataLoggerComponent` (registro de métricas)

### 2.2 Estruturas de dados

Crie uma `USTRUCT` para parâmetros de ambiente:

- `Pressure_Pa`
- `Temperature_C`
- `Salinity_PSU`
- `Density_kg_m3` (calculada ou override)
- `DynamicViscosity_Pa_s`
- `FlowVelocity_m_s`

E outra para parâmetros da hélice:

- `Diameter_m`
- `Pitch_m`
- `BladeCount`
- `AreaRatio`
- `RPM`

### 2.3 Curvas e tabelas

Use `DataTable`/`CurveFloat` para coeficientes hidrodinâmicos:

- `KT(J)` coeficiente de empuxo
- `KQ(J)` coeficiente de torque

com

\[
J = \frac{V_A}{nD}
\]

onde:
- \(V_A\): velocidade de avanço
- \(n\): rotações por segundo
- \(D\): diâmetro

---

## 3) Modelo físico mínimo viável (tempo real)

Com os coeficientes empíricos:

\[
T = K_T(J)\,\rho\,n^2\,D^4
\]
\[
Q = K_Q(J)\,\rho\,n^2\,D^5
\]
\[
P_{shaft} = 2\pi n Q
\]

### 3.1 Cavitação (estimativa)

Número de cavitação:

\[
\sigma = \frac{p_\infty - p_v}{\frac{1}{2}\rho V_{tip}^2}
\]

- \(p_\infty\): pressão local
- \(p_v\): pressão de vapor (função da temperatura)
- \(V_{tip} = \pi D n\)

Quando \(\sigma\) cai abaixo de limiar experimental para a hélice, marque “risco de cavitação”.

---

## 4) Dependências recomendadas

- **Unreal Engine 5.3+**
- **Chaos Physics** (nativo)
- **Niagara** para visualização de vórtices/cavitação
- **Python plugin** (opcional, para calibração)
- **CSV/JSON exporter** para análise externa

Para alta fidelidade:
- OpenFOAM (externo) + importação de resultados para LUT no Unreal.

---

## 5) Pipeline sugerido

1. **Construir cena base** (tanque de água + hélice + câmera + UI).
2. **Implementar física reduzida** (`UPropellerHydroComponent`).
3. **Criar painel de parâmetros** (UMG): sliders para RPM, T, salinidade etc.
4. **Logar métricas** por frame/timestep (CSV).
5. **Validar com casos conhecidos** (curvas KT/KQ do fabricante).
6. **Calibrar** coeficientes para seu regime operacional.
7. **Adicionar visualização** de campos (streamlines aproximadas, cavitação).

---

## 6) Exemplo de lógica C++ (pseudocódigo)

```cpp
void UPropellerHydroComponent::TickComponent(float Dt)
{
    const float n = RPM / 60.0f;
    const float J = AdvanceVelocity / FMath::Max(0.01f, n * Diameter);

    const float KT = KT_Curve->Eval(J);
    const float KQ = KQ_Curve->Eval(J);

    const float T = KT * Density * n*n * FMath::Pow(Diameter, 4);
    const float Q = KQ * Density * n*n * FMath::Pow(Diameter, 5);
    const float ShaftPower = 2.0f * PI * n * Q;

    ApplyAxialForce(T);
    ApplyShaftTorque(Q);

    const float Vtip = PI * Diameter * n;
    const float Pv = VaporPressureFromTemperature(TemperatureC);
    const float Sigma = (Pressure - Pv) / (0.5f * Density * Vtip * Vtip + 1e-6f);

    CavitationRisk = (Sigma < CavitationSigmaThreshold);
    LogFrame(T, Q, ShaftPower, Sigma, J);
}
```

---

## 7) Modelagem da água (densidade e viscosidade)

Você pode usar:

- **Modelo simplificado:** densidade fixa (ex.: 1025 kg/m³ para água do mar).
- **Modelo dependente de T/S/P:** equação UNESCO/TEOS-10 (melhor realismo).

Estratégia boa para tempo real:

- Pré-calcular uma grade 3D (T, S, P) -> (ρ, μ)
- Carregar como tabela e interpolar linearmente em runtime.

---

## 8) Validação (essencial)

Valide em etapas:

1. **Unitário de fórmulas:** conferir dimensão/unidade de T, Q, P.
2. **Comparação com literatura/fabricante:** curvas KT/KQ.
3. **Teste de sensibilidade:** variar um parâmetro por vez.
4. **Convergência temporal:** reduzir timestep e comparar métricas.

KPIs sugeridos:
- Erro percentual de empuxo/torque contra referência.
- Estabilidade numérica para faixa de RPM.
- Repetibilidade de resultados com mesma seed.

---

## 9) Limitações esperadas

- Unreal não substitui CFD de malha fina para fenômenos complexos 3D.
- Cavitação real envolve bolhas, colapso e acústica complexos.
- Turbulência avançada requer solver dedicado.

Use o sistema para exploração de cenários e engenharia preliminar.

---

## 10) Roadmap de implementação (4 semanas)

### Semana 1
- Estrutura de projeto, ator da hélice, UI básica.
- Parâmetros ambientais e logging.

### Semana 2
- Implementação KT/KQ e cálculo de empuxo/torque.
- Curvas por DataTable + testes sintéticos.

### Semana 3
- Cavitação estimada e efeitos visuais (Niagara).
- Exportação de resultados para Python.

### Semana 4
- Calibração com dados reais/CFD.
- Dashboard de análise e relatório final.

---

## 11) Próximos passos para você executar agora

1. Criar projeto UE5 (C++).
2. Implementar `UPropellerHydroComponent` com as fórmulas acima.
3. Montar UI de sliders (RPM, T, S, P).
4. Adicionar DataTable de curvas `KT(J)` e `KQ(J)`.
5. Rodar cenários e exportar CSV para validação.

Se quiser, o próximo passo pode ser gerar um **esqueleto de classes C++ pronto** (headers + cpp) para colar no seu projeto Unreal.
