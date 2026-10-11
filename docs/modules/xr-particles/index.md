# xrParticles

## Порционы

### 4. Noise и связка с движком

Работа с шумом и интеграция с xrEngine:
- `particle_noise.h`/`particle_noise.cpp` - основные классы для генерации шума
- `particle_noise_*.h`/`.cpp` - реализация конкретных шумовых алгоритмов
- Интеграция с xrEngine через `PAPI::ParticleManager`

См. также:
- [PAPI и ядро симуляции](modules/xr-particles/core.md)
- [Actions: ядро и домены](modules/xr-particles/actions.md)
