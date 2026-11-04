# ganipedia-gitops

Repo ini isinya **cuma manifest Kubernetes**, bukan source code aplikasi.
Dipakai ArgoCD (via ApplicationSet) untuk auto-deploy tiap app yang ada di
folder `apps/`.

## Struktur

```
apps/
  <nama-app>/
    deploy.yaml   <- semua manifest K8s untuk app itu (Namespace, Deployment,
                     Service, Ingress, HPA, PDB, Quota, LimitRange, NetworkPolicy)
```

## Cara kerja

1. ArgoCD ApplicationSet scan folder `apps/*`
2. Tiap subfolder otomatis jadi 1 ArgoCD Application
3. Nambah app baru = nambah 1 folder baru di sini, commit, push
4. Jenkins (CI) HANYA boleh punya akses push ke repo INI, bukan ke repo kode
   aplikasi — jadi kalau mau update image tag setelah build, Jenkins commit
   perubahan `image:` di `apps/<nama-app>/deploy.yaml` di sini.

## Jangan taruh di sini

- Source code aplikasi (tetap di repo masing-masing: pare-moments, api-saku, dst)
- Secret/credential asli (pakai Vault, bukan hardcode di YAML)
