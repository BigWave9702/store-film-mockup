```mermaid

erDiagram
    direction LR
    
    nha_xuat_ban ||--o{ chien_dich : "chay"
    nha_xuat_ban ||--o{ quy_tac_hoa_hong : "co"
    nha_xuat_ban ||--o{ giao_dich_hoa_hong : "nhan"
    nha_xuat_ban ||--o{ gan_ket_nguon : "duoc_quy_gan"
    nha_xuat_ban ||--o{ chuyen_doi : "duoc_ghi_nhan"
    nha_xuat_ban ||--o{ doi_soat_hoa_hong : "doi_soat"
    nha_xuat_ban ||--o{ chi_tra : "nhan_chi_tra"

    nha_xuat_ban||--o{ gan_chien_dich_nha_xuat_ban : "duoc_gan"
    chien_dich ||--o{ gan_chien_dich_nha_xuat_ban : "co"
    chien_dich ||--o{ chien_dich_noi_dung : "co"

    gan_chien_dich_nha_xuat_ban ||--o{ lien_ket_theo_doi : "sinh"
    lien_ket_theo_doi ||--o{ luot_nhap : "ghi_nhan"
    lien_ket_theo_doi ||--o{ luot_xem : "ghi_nhan"
   
    gan_chien_dich_nha_xuat_ban ||--o{ gan_ket_nguon : "quy_gan"
    lien_ket_theo_doi ||--o{ gan_ket_nguon : "nguon_first_click"
    gan_ket_nguon ||--o{ chuyen_doi : "dan_toi"

    quy_tac_hoa_hong ||--o{ giao_dich_hoa_hong : "ap_dung"
    chuyen_doi ||--o{ giao_dich_hoa_hong : "phat_sinh"
    chuyen_doi ||--o{ dieu_chinh_hoa_hong : "bi_dieu_chinh"
    giao_dich_hoa_hong ||--o{ dieu_chinh_hoa_hong : "dieu_chinh"

    ky_doi_soat ||--o{ doi_soat_hoa_hong : "gom"
    doi_soat_hoa_hong ||--o{ giao_dich_hoa_hong : "bao_gom"
    doi_soat_hoa_hong ||--o{ chi_tra : "chi_tra"

    giao_dich_hoa_hong ||--o{ chi_tiet_doi_soat : "duoc_doi_soat"
    dieu_chinh_hoa_hong ||--o{ chi_tiet_doi_soat : "duoc_doi_soat"
    doi_soat_hoa_hong ||--o{ chi_tiet_doi_soat : "gom_chi_tiet"

    nha_xuat_ban {
        BIGINT id PK
        VARCHAR email UK
        VARCHAR mat_khau_hash
        VARCHAR ten
        VARCHAR lien_he
        SMALLINT trang_thai
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    chien_dich {
        BIGINT id PK
        VARCHAR ma_chien_dich UK
        VARCHAR ten
        TIMESTAMPTZ bat_dau_luc
        TIMESTAMPTZ ket_thuc_luc
        SMALLINT trang_thai
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }
 
    gan_chien_dich_nha_xuat_ban {
        BIGINT id PK
        BIGINT ma_chien_dich FK
        BIGINT ma_nha_xuat_ban FK
        TIMESTAMPTZ gan_luc
        TIMESTAMPTZ huy_luc
        SMALLINT trang_thai
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
    }

    chien_dich_noi_dung {
       BIGINT id PK
       VARCHAR ma_chien_dich UK
       BIGINT ma_noi_dung
       SMALLINT trang_thai
       TIMESTAMPTZ ngay_tao
       TIMESTAMPTZ ngay_cap_nhat
       BIGINT nguoi_tao
    }

    lien_ket_theo_doi {
        BIGINT id PK
        BIGINT ma_gan_chien_dich_nha_xuat_ban FK
        BIGINT ma_noi_dung
        VARCHAR ma_theo_doi UK
        VARCHAR url_dich
        VARCHAR nguon
        SMALLINT trang_thai
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }   
    
    gan_ket_nguon {
        BIGINT id PK
        VARCHAR ma_an_danh UK
        BIGINT ma_nguoi_dung
        BIGINT ma_gan_chien_dich_nha_xuat_ban FK
        BIGINT ma_lien_ket FK
        VARCHAR ma_theo_doi
        BIGINT ma_nha_xuat_ban FK
        BIGINT ma_chien_dich
        VARCHAR nguon_dau
        VARCHAR mo_hinh_quy_gan
        TIMESTAMPTZ quy_gan_luc
        TIMESTAMPTZ het_han_luc
    }

    luot_nhap {
        BIGINT id PK
        BIGINT ma_lien_ket FK
        VARCHAR ma_an_danh
        VARCHAR dia_chi_ip
        VARCHAR user_agent
        VARCHAR nguon
        TIMESTAMPTZ nhap_luc
    }
    
    luot_xem {
        BIGINT id PK
        BIGINT ma_lien_ket FK
        BIGINT ma_nguoi_dung
        BIGINT ma_noi_dung
        VARCHAR ma_an_danh
        INTEGER thoi_gian_xem
        NUMERIC ty_le_xem
        TIMESTAMPTZ xem_luc
        VARCHAR ma_su_kien_nguon UK
    } 

    chuyen_doi {
        BIGINT id PK
        BIGINT ma_gan_ket FK
        BIGINT ma_nha_xuat_ban FK
        BIGINT ma_chien_dich FK
        BIGINT ma_lien_ket FK
        VARCHAR ma_theo_doi
        VARCHAR ma_an_danh
        BIGINT ma_nguoi_dung
        BIGINT ma_dang_ky
        BIGINT ma_goi_cuoc
        BIGINT ma_thanh_toan UK
        VARCHAR ma_giao_dich_cong
        VARCHAR ma_su_kien_nguon UK
        SMALLINT loai_giao_dich
        NUMERIC so_tien_gross
        NUMERIC so_tien_giam_gia
        NUMERIC so_tien_thue
        NUMERIC so_tien_net
        VARCHAR don_vi_tien
        TIMESTAMPTZ chuyen_doi_luc
        SMALLINT trang_thai
        TIMESTAMPTZ ngay_tao
    }

    quy_tac_hoa_hong {
        BIGINT id PK
        BIGINT ma_chien_dich FK
        BIGINT ma_nha_xuat_ban FK
        SMALLINT loai_su_kien
        SMALLINT loai_tinh
        NUMERIC gia_tri
        SMALLINT co_so_tinh
        VARCHAR don_vi_tien
        INTEGER gioi_han_so_luong
        NUMERIC gioi_han_so_lan
        INTEGER thoi_gian_xem_toi_thieu
        NUMERIC ty_le_xem_toi_thieu
        TIMESTAMPTZ hieu_luc_tu
        TIMESTAMPTZ hieu_luc_den
        SMALLINT trang_thai
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    giao_dich_hoa_hong {
        BIGINT id PK
        BIGINT ma_nha_xuat_ban FK
        BIGINT ma_chuyen_doi FK
        BIGINT ma_quy_tac_hoa_hong FK
        SMALLINT loai_su_kien
        SMALLINT loai_tinh
        SMALLINT co_so_tinh
        NUMERIC so_tien_co_so
        NUMERIC gia_tri_tinh
        NUMERIC so_tien_hoa_hong
        VARCHAR don_vi_tien
        SMALLINT trang_thai
        VARCHAR ky_doi_soat
        BIGINT ma_doi_soat_hoa_hong FK
        BIGINT nguoi_duyet
        TIMESTAMPTZ duyet_luc
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
    }

    dieu_chinh_hoa_hong {
        BIGINT id PK
        BIGINT ma_giao_dich_hoa_hong FK
        BIGINT ma_chuyen_doi FK
        SMALLINT loai
        NUMERIC so_tien
        VARCHAR don_vi_tien
        VARCHAR ly_do
        VARCHAR ma_tham_chieu
        TIMESTAMPTZ ngay_tao
        BIGINT nguoi_tao
    }

    ky_doi_soat {
        BIGINT id PK
        VARCHAR ma_ky UK
        TIMESTAMPTZ tu_ngay
        TIMESTAMPTZ den_ngay
        SMALLINT trang_thai
        TIMESTAMPTZ ngay_khoa
        BIGINT nguoi_khoa
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
    }

    doi_soat_hoa_hong {
        BIGINT id PK
        BIGINT ma_ky_doi_soat FK
        BIGINT ma_nha_xuat_ban FK
        NUMERIC tong_gia_tri_giao_dich
        NUMERIC tong_hoa_hong
        NUMERIC tong_dieu_chinh
        NUMERIC so_tien_phai_tra
        VARCHAR don_vi_tien
        SMALLINT trang_thai
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
    }

    chi_tiet_doi_soat {
        BIGINT id PK
        BIGINT ma_doi_soat_hoa_hong FK
        BIGINT ma_giao_dich_hoa_hong FK
        BIGINT ma_dieu_chinh_hoa_hong FK
        SMALLINT loai
        NUMERIC so_tien_hoa_hong
        NUMERIC so_tien_dieu_chinh
        NUMERIC so_tien_thanh_toan
        SMALLINT trang_thai
        VARCHAR ly_do
        TIMESTAMPTZ ngay_tao
        BIGINT nguoi_tao
    }

    chi_tra {
        BIGINT id PK
        BIGINT ma_doi_soat_hoa_hong FK
        BIGINT ma_nha_xuat_ban FK
        VARCHAR ma_phuong_thuc
        NUMERIC so_tien
        VARCHAR don_vi_tien
        VARCHAR ma_giao_dich_chi_tra
        SMALLINT trang_thai
        TIMESTAMPTZ yeu_cau_luc
        TIMESTAMPTZ chi_tra_luc
        VARCHAR ly_do_that_bai
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
    }
```