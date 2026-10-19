```mermaid
erDiagram

    ngon_ngu ||--o{ noi_dung_da_ngon_ngu : "co"
    ngon_ngu ||--o{ danh_muc_da_ngon_ngu : "co"
    ngon_ngu ||--o{ phu_de : "co"
    ngon_ngu ||--o{ anh_bia : "co"
    ngon_ngu ||--o{ thong_tin_noi_dung : "la_ngon_ngu_goc"

    nha_cung_cap_noi_dung ||--o{ noi_dung : "cung_cap"
    nha_cung_cap_noi_dung ||--o{ tai_khoan_he_thong : "so_huu"

    noi_dung ||--o{ noi_dung_da_ngon_ngu : "da_ngon_ngu"
    noi_dung ||--o{ phien_ban_noi_dung : "co_phien_ban"
    noi_dung ||--|| thong_tin_noi_dung : "co_thong_tin"
    noi_dung ||--o{ mua_phim : "co_mua"
    noi_dung ||--o{ noi_dung_danh_muc : "gan_danh_muc"
    noi_dung ||--o{ phu_de : "co_phu_de"
    noi_dung ||--o{ anh_bia : "co_anh"
    noi_dung ||--|| quy_trinh_noi_dung : "co_quy_trinh"
    noi_dung ||--o{ lich_su_quy_trinh : "co_lich_su"
    noi_dung ||--o{ lich_phat_hanh : "co_lich_phat"
    noi_dung ||--o{ ban_quyen_noi_dung : "co_ban_quyen"

    noi_dung ||--o{ noi_dung : "cha_con"
    mua_phim ||--o{ noi_dung : "chua_tap"

   cap_quyen_xem ||--o{ chinh_sach_quyen_xem : "áp dụng"
   noi_dung ||--o{ chinh_sach_quyen_xem : "có chính sách"

    phien_ban_noi_dung }o--o| noi_dung : "thuoc"
    noi_dung }o--o| phien_ban_noi_dung : "mac_dinh"

    danh_muc ||--o{ danh_muc_da_ngon_ngu : "da_ngon_ngu"
    danh_muc ||--o{ noi_dung_danh_muc : "duoc_gan"
    danh_muc ||--o{ danh_muc : "cha_con"

    dao_dien ||--o{ noi_dung_dao_dien : "tham_gia"
    noi_dung ||--o{ noi_dung_dao_dien : "co_dao_dien"

    dien_vien ||--o{ noi_dung_dien_vien : "tham_gia"
    noi_dung ||--o{ noi_dung_dien_vien : "co_dien_vien"

    tai_khoan_he_thong ||--o{ tai_khoan_vai_tro : "duoc_gan"
    vai_tro ||--o{ tai_khoan_vai_tro : "gan_cho"
    vai_tro ||--o{ vai_tro_quyen_han : "co"
    quyen_han ||--o{ vai_tro_quyen_han : "duoc_cap"

    tai_khoan_he_thong ||--o{ quy_trinh_noi_dung : "phu_trach"
    tai_khoan_he_thong ||--o{ lich_su_quy_trinh : "thuc_hien"

    tai_khoan_he_thong {
        BIGINT id PK
        VARCHAR email UK
        VARCHAR mat_khau_hash
        VARCHAR ho_ten
        SMALLINT loai
        BIGINT ma_nha_cung_cap FK
        SMALLINT trang_thai
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    vai_tro {
        BIGINT id PK
        VARCHAR ma_vai_tro UK
        VARCHAR ten
		VARCHAR duong_dan
		SMALLINT la_menu
		BIGINT ma_quyen_cha
        VARCHAR mo_ta
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    quyen_han {
        BIGINT id PK
        VARCHAR ma_quyen UK
        VARCHAR ten
        VARCHAR mo_ta
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    tai_khoan_vai_tro {
        BIGINT id PK
        BIGINT ma_tai_khoan_he_thong FK
        BIGINT ma_vai_tro FK
        TIMESTAMPTZ ngay_tao
        BIGINT nguoi_tao
    }

    vai_tro_quyen_han {
        BIGINT id PK
        BIGINT ma_vai_tro FK
        BIGINT ma_quyen_han FK
        TIMESTAMPTZ ngay_tao
        BIGINT nguoi_tao
    }

    nha_cung_cap_noi_dung {
        BIGINT id PK
        VARCHAR ten
        VARCHAR loai
        VARCHAR lien_he
        SMALLINT trang_thai
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    ngon_ngu {
        BIGINT id PK
        VARCHAR ma_ngon_ngu UK
        VARCHAR ten
        SMALLINT trang_thai
        SMALLINT thu_tu
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    noi_dung {
        BIGINT id PK
        BIGINT ma_nha_cung_cap FK
        VARCHAR tieu_de
        TEXT mo_ta
        SMALLINT loai
        BIGINT ma_noi_dung_cha FK
        BIGINT ma_mua_phim FK
        INT so_tap
        SMALLINT trang_thai
        BIGINT ma_phien_ban_mac_dinh FK
        SMALLINT nam_phat_hanh
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    mua_phim {
        BIGINT id PK
        BIGINT ma_noi_dung FK
        INT so_mua
        VARCHAR tieu_de
        TEXT mo_ta
        SMALLINT nam_phat_hanh
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    noi_dung_da_ngon_ngu {
        BIGINT id PK
        BIGINT ma_noi_dung FK
        BIGINT ma_ngon_ngu FK
        VARCHAR tieu_de
        VARCHAR mo_ta_ngan
        TEXT mo_ta
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    thong_tin_noi_dung {
        BIGINT id PK
        BIGINT ma_noi_dung FK
        VARCHAR quoc_gia
        BIGINT ma_ngon_ngu_goc FK
        JSONB the_loai_tag
        INT thoi_luong_phut
        SMALLINT do_tuoi
        VARCHAR xep_hang
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    dao_dien {
        BIGINT id PK
        VARCHAR ho_ten
        VARCHAR ten_khac
        VARCHAR quoc_tich
        DATE ngay_sinh
        TEXT gioi_thieu
        VARCHAR anh_dai_dien
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    dien_vien {
        BIGINT id PK
        VARCHAR ho_ten
        VARCHAR ten_khac
        VARCHAR quoc_tich
        DATE ngay_sinh
        TEXT gioi_thieu
        VARCHAR anh_dai_dien
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    noi_dung_dao_dien {
        BIGINT id PK
        BIGINT ma_noi_dung FK
        BIGINT ma_dao_dien FK
        SMALLINT thu_tu
        TIMESTAMPTZ ngay_tao
        BIGINT nguoi_tao
    }

    noi_dung_dien_vien {
        BIGINT id PK
        BIGINT ma_noi_dung FK
        BIGINT ma_dien_vien FK
        SMALLINT thu_tu
        VARCHAR vai_dien
        TIMESTAMPTZ ngay_tao
        BOOLEAN da_xoa
    }

    phien_ban_noi_dung {
        BIGINT id PK
        BIGINT ma_noi_dung FK
        INT so_phien_ban
        BIGINT ma_tai_nguyen_media
        VARCHAR ghi_chu
        TIMESTAMPTZ ngay_tao
        BIGINT nguoi_tao
    }

    danh_muc {
        BIGINT id PK
        VARCHAR ten
        TEXT mo_ta
        BIGINT ma_danh_muc_cha FK
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    danh_muc_da_ngon_ngu {
        BIGINT id PK
        BIGINT ma_danh_muc FK
        BIGINT ma_ngon_ngu FK
        VARCHAR ten
        TEXT mo_ta
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    noi_dung_danh_muc {
        BIGINT id PK
        BIGINT ma_noi_dung FK
        BIGINT ma_danh_muc FK
        TIMESTAMPTZ ngay_tao
        BIGINT nguoi_tao
    }

    phu_de {
        BIGINT id PK
        BIGINT ma_noi_dung FK
        BIGINT ma_ngon_ngu FK
        VARCHAR khoa_tep
        VARCHAR dinh_dang
        VARCHAR loai_phu_de
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    anh_bia {
        BIGINT id PK
        BIGINT ma_noi_dung FK
        SMALLINT loai
        VARCHAR khoa_tep
        VARCHAR ten_tep
        VARCHAR mime_type
        BIGINT kich_thuoc
        INT chieu_rong
        INT chieu_cao
        VARCHAR checksum
        BIGINT ma_ngon_ngu FK
        BOOLEAN la_mac_dinh
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    quy_trinh_noi_dung {
        BIGINT id PK
        BIGINT ma_noi_dung FK
        VARCHAR buoc
        SMALLINT trang_thai
        BIGINT ma_nguoi_phu_trach FK
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
    }

    lich_su_quy_trinh {
        BIGINT id PK
        BIGINT ma_noi_dung FK
        VARCHAR buoc_tu
        VARCHAR buoc_den
        BIGINT ma_nguoi_thuc_hien FK
        TIMESTAMPTZ thoi_diem
        VARCHAR ghi_chu
    }

    lich_phat_hanh {
        BIGINT id PK
        BIGINT ma_noi_dung FK
        TIMESTAMPTZ phat_hanh_luc
        TIMESTAMPTZ het_hieu_luc_luc
        SMALLINT trang_thai
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }

    ban_quyen_noi_dung {
        BIGINT id PK
        BIGINT ma_noi_dung FK
        TIMESTAMPTZ bat_dau_luc
        TIMESTAMPTZ ket_thuc_luc
        JSONB lanh_tho
        TIMESTAMPTZ ngay_tao
        TIMESTAMPTZ ngay_cap_nhat
        BIGINT nguoi_tao
        BOOLEAN da_xoa
    }
	
	cap_quyen_xem {
		BIGINT id PK
		VARCHAR(50) ma_cap_quyen UK
		SMALLINT thu_tu UK
		SMALLINT trang_thai
		TIMESTAMPTZ ngay_tao
		TIMESTAMPTZ ngay_cap_nhat
		BIGINT nguoi_tao
		BOOLEAN da_xoa
	}

	chinh_sach_quyen_xem {
		BIGINT id PK
		BIGINT ma_noi_dung FK
		BIGINT ma_cap_quyen FK
		BOOLEAN duoc_xem
		VARCHAR(20) chat_luong_toi_da
		INT so_tap_toi_da
		TIMESTAMPTZ ngay_bat_dau
		TIMESTAMPTZ ngay_ket_thuc
		SMALLINT trang_thai
		TIMESTAMPTZ ngay_tao
		TIMESTAMPTZ ngay_cap_nhat
		BIGINT nguoi_tao
		BOOLEAN da_xoa
	}
```